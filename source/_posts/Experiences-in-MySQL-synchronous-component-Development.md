---
title: Experiences in MySQL synchronous component Development
date: 2024-09-02 14:45:49
tags:
---
# Background
Due to single table performance limitations in MySQL, sharding techniques are widely used. For example, if an order table has 100,000 daily transactions, it would accumulate over 30 million records in less than a year, which is a bottleneck for MySQL single table performance. We would typically split the order table into 100 sub tables based on a certain rule(such as user_id), to distribute the order data as evenly as possible across these 100 tables. (The evenness of distribution depends on how uniformly your user_ids are generated. For instance, in my previous company, due to a bug in the user_id generation code, even-numbered user_id is more than odd-numbered, as a result, in tables with even-numbered indices containing significantly more data than those with odd-numbered indices for both order and transaction tables.)

So, what's the cost?  

After implementing table sharding, it becomes challenging to view orders from a global perspective. For example, if you want to see the total order volume for a particular day, you'll find that you need to search all 100 order tables to get the information. When a user reports an issue with a specific order, you have to obtain the user_id from them; otherwise, you can't determine which table the order is in. These are the inconveniences brought about by database and table sharding techniques.

To address these issues, we often use heterogeneous databases as a solution. We create a mirror in another database. High-frequency user requests are handled by MySQL, while low-frequency requests that require a global view are directed to the mirror for querying.

This naturally raises two questions. First, which database should take on the responsibility of being this mirror?  

# Elasticsearch
In practice, we chose Elasticsearch. This decision was mainly based on successful implementations in other companies at the time. In contrast, Cassandra was not popular in China then, and the same was true for PostgreSQL and MongoDB.

Using Elasticsearch as a mirror is not without its problems. Firstly, for some basic data types, Elasticsearch doesn't have fields that can be directly compared with MySQL.  

For transaction systems, the accuracy of monetary amounts is crucial. In MySQL, amounts are typically stored using `decimal` fields. However, Elasticsearch doesn't have a corresponding decimal field. For numbers after the decimal point, there are only two options: store them as strings or as floating-point numbers.

The drawbacks of floating-point numbers are well-known in computer science fundamentals. They are only an approximation of the actual value. For example, what you see as 12.01 might actually be stored as 12.009999999999 or 12.0111111111 in the computer. If you use floating-point numbers for addition, subtraction, or multiplication, the calculation errors can become significant after just a few operations.

For the amount stored in a string, the biggest problem is that you can't use range searches. For example, if you want to search for orders from yesterday with an amount greater than 100, it's difficult with strings because the comparison is based on ASCII codes, not numerical values.

This also brings up another drawback of Elasticsearch strings. Elasticsearch was originally developed as a search engine, and its string fields have tokenization and similarity features. For example, if there's an order with the number `O8135ZX7343`, Elasticsearch might split this into two keywords, `O8135` and `ZX7343`, for searching and storing. Or it might store this order as two different inverted indices. This needs to be avoided by adding the `'keyword'` type when querying and creating tables.

Besides these, there are other issues. For instance, MySQL might rename field names or modify field order (fortunately, DBAs in large companies usually prohibit such operations, as they can also cause field order anomalies in slave databases) or change field types (e.g., from `integer` to `varchar` - I'm not sure if DBAs in large companies allow this type of operation). However, once a field is established in Elasticsearch, it cannot be modified, such as changing its type or name.

Here we have to raise a questiont: operations like DDL in Mysql often need to be performed in Elasticsearch as well. For example, if a table is created in MySQL, a corresponding index needs to be manually created in the Elasticsearch mirror database. If a field is added to a MySQL table, the same needs to be done in Elasticsearch. If not done manually, two situations can occur:

1. Elasticsearch prohibits creating fields on its own and will report an error and log it.
2. Elasticsearch will guess the field type based on the received data. For example, if it receives 1, 2, 3, it might guess the field type as int; if it receives 3.34, it might guess float. This can easily lead to incorrect field type guesses, and as mentioned earlier, once a field type is created, it can't be modified, which can cause significant operational and maintenance issues.

After explaining these pros and cons, let's look at how to specifically synchronize data from MySQL to Elasticsearch.

# Approach 1: Dual Write
When data arrives, it's first written to MySQL, and after successful insertion, it's then written to Elasticsearch.  
Disadvantages: There's a chance of write failure, requiring compensation scripts. It has requirements on write order and significantly impacts the business, as every business process that needs dual-table writing needs to write to Elasticsearch.

# Approach 2: Using compoment for writing, without developer effect, automatically synchronize data to Elasticsearch
We researched several Approaches, considering the following points:

1. Activity: Those with only single-digit attention or never-raised issues can be directly disregarded.
2. Usability: As is well known, many open-source products may become unusable due to version iterations, such as requiring specific environments, or may even be unusable when first open-sourced.
3. Reliability of principle: It should ensure that the data in MySQL and Elasticsearch corresponds one-to-one, without excess or deficit. This requires you to read the source code and implementation process after successful deployment to judge.
4. Compatibility with company tech stack: Whether additional machine resources need to be applied for; what the maintenance cost is like.

At that time point (2018), after considering these factors, we selected a few compoment from open-source:

## [Logstash](https://github.com/logstash-plugins/logstash-input-jdbc)
**Principle**: Periodically scans the entire table, extracts data from MySQL, and writes it to Elasticsearch.

**Disadvantage**: 
1. Not feasible for tables with very large data volumes; poor real-time performance.
 
## [Canal](https://github.com/alibaba/canal)
**Principle**: By disguising itself as a MySQL slave, it listens to MySQL binlog and converts binlog events into internally defined message events in Canal. You can directly develop plugins and compile them into JAR packages for use, or Canal can be configured to send message events to Kafka for consumption.  

**Advantages**:
1. Produced by a major tech company; quality assured.
     
**Disadvantages**:
1. At that time in 2019, Canal couldn't solve the problem of how to import existing data in the database into Elasticsearch, requiring self-development to resolve. This was the main reason we didn't choose it. Even now, I personally consider its approach to handling existing data quite crude.
2. Java technology stack, which didn't align with our company's tech stack.
3. This component is divided into multiple services, each with various configurations, making maintenance costly. At that time, and for a considerable period afterward, I was solely responsible for this aspect of work in our team.

## [go-mysql-elasticsearch](https://github.com/go-mysql-org/go-mysql-elasticsearch)
**Principle**: At any given moment, we can divide the data in MySQL into existing  and incremental data. For existing data, we use the mysqldump with `--single-transaction` and  `--master-data` to export it as text. This command produces content as follows and additionally generates a position, which is the starting point for consuming the binlog.  

Then, it gradually parses the content in the text and writes it to Elasticsearch. After the existing text data is written, it actively pulls and consumes data from the binlog.  

This principle is very reliable. By consulting MySQL official documentation and the company's DBA, it's the standard process for establishing a MySQL slave, indicating that the construction process is theoretically sound.  
It's worth mentioning that the author at that time was the CTO of TiDB.

**Advantages**:
1. Consistency between existing and incremental data is guaranteed; and the operation is simple.
2. The tech stack aligns with the company's, making it convenient for colleagues to maintain together.

**Disadvantages**:

1. It doesn't support incrementally adding data tables. For example, if you initially synchronize the order table, and then the account table also needs to be sharded and synchronized to Elasticsearch, this tool would require starting a new service, importing the corresponding existing data for that table, consuming the existing text to write to Elasticsearch, and continuously monitoring the binlog and writing to Elasticsearch again.
2. It uses binlog instead of GTID as a tool to track incremental positions. This isn't a particularly serious problem; it's just that when machines or upstream databases are migrated, the binlog position needs to be changed. Unfortunately, our company frequently migrated data centers and databases for various reasons.
3. When there are a lot of upstream binlogs, it can cause severe blocking. For example, with a billing table, when users are on their billing date or overdue date, there will be large-scale data changes, generating a large number of binlogs upstream and pushing them to go-mysql-elasticsearch. The writing performance of this component is limited (actually, it's Elasticsearch's writing performance that's limited), leading to delays in a large number of synchronizations. (This is because consuming binlog needs to be in order, for example, for the same piece of data, if it's first inserted then updated, your binlog consumption must follow this order, you absolutely cannot update first then insert. Of course, this only applies to the same table; if it's data from different tables, this problem doesn't exist. This is also why [Canal](https://github.com/alibaba/canal), mentioned earlier, supports converting binlog messages into internal messages and throwing them into Kafka: if produced to Kafka, consumers can be increased, for example, consumers for each table can consume independently, avoiding the situation where too many binlogs produced by the billing table cause blocking of binlog synchronization for the order and user tables as well.)
4. Some higher database permissions are required, because it involves commands such as `master-data` and `--single-transaction`, so you need to apply and explain to the DBA for an account with `RELOAD` AND `REPLICATION CLIENT` permissions
5. Because it involves the operation of mysqldump commder, it's forbiden not operate it in the master database. It is best to operate it in the BI database(BI database here refers to the slave database specially built for big data teams. Some professional teams or large companies will build such slave databases. If not, you can choose a slave database with less load). However,The problems that are occur here are:
   1. When multiple people and teams are using mysqldump, they often block each other's dump threads, which often occurs in BI databases (big data teams also often dump)
   2. The implementation of the original code is not efficient enough. In a nutshell, it used golang's `io.pipe` function, which is implemented as an unbuffered channel.
     	```go
		func Pipe() (*PipeReader, *PipeWriter) {
		    p := &pipe{
		        wrCh: make(chan []byte),
		        rdCh: make(chan int),
		        done: make(chan struct{}),

		    }
		    return &PipeReader{p}, &PipeWriter{p}
		}
     	```	
		This means that when the sender sends data to the channel, it will be blocked immediately until the receiver reads the data from the channel. In other words, when it reads a byte of data from mysqldump, it will immediately hand over the CPU scheduling until the goroutine of its corresponding reading process reads a byte of data from the other end of the queue, which will cause frequent context switching. In fact, the initial observation is that the import of inventory data and consumption is extremely slow, and there are often cases where the waiting time process is killed by the mysql server.
		
		The solution at that time was to rewrite the logic of this part. The goroutine that writes the file exports the output to the local file in Linux, and the goroutine that parses the file reads the file. This avoids the context switching of the io.pipe read and write goroutines.

		However, this is still not efficient enough. The ultimate solution is to manually execute the same command to export the file to the local Linux, and then have a program to fetch and parse the file and write it to Elasticsearch.
		

6. Lacking of monitoring, and no pre-embedded statistical interfaces for monitoring systems.


# Reflection
Finally, in the process of practicing go-mysql-elasticsearch, we encountered all the above problems. So, somebody may ask: Why did you choose a infrastructure component with so many problems? At first, I admire this component very much. It is easy to use and the code is clear and easy to understand. As a user of open source projects, if you are not satisfied with something, you can actually rewrite it and push it back.

Looking back, It should have been replaced with a platform-based Canal at some point (perhaps in 2021).However, at that time, I was already burned out by another infrastructure component. The platform-based compoment should have more time and more developer to support it(not only this one). 

It shouldn't use the time managerment of developing product requirement to manage the time of developing infrastructure components. Because if the product requirements are clear, the workload is also determined. However, the time for infrastructure component selection and secondary development is often vague, because it is impossible to research it very clearly at the beginning (it will take a long time), and you can only have a rough assessment. Then, in the actual development, you will often encounter some unexpected challenges. This leads to insufficient time.

Anyway, I am still very grateful for this experience. It is the first time for me to independently undertake a infrastructure component,understanding the background and learn how to select technology, research, secondary developing, and promotion to other colleagues and maintenance.

