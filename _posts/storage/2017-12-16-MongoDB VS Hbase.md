---
layout: post
title: MongoDB VS Hbase
category: 存储系统
tags: mongodb
---

## 流行度##
2017年12月，db-engines上排名Mongodb 第5,Hbase第15。

## 比较概述 ##

比较项 | Hbase | Mongodb
---   | --- |  ---
数据库模型   | 宽列数据库 | 文档数据库
最早发布版本  | 2008     | 2009
实现语言     | Java      | C++
支持平台     | 主要linux | linux、windows
分区方法     | Sharding | Sharding
Replication methods | selectable replication factor	 | 主从方式
MapReduce支持 | 支持    | 支持
与Spark集成    | 支持    | 支持

## 成熟度 ##
初次发布版本时间都差不多，至2017/12/13，github上，Mongodb发布版本489次，有322位代码贡献者，Hbase发布版本568次，162位代码贡献者。

## 架构 ##
Hbase模块构成：

<img src="https://www.ibm.com/developerworks/cn/analytics/library/ba-cn-bigdata-hbase/image002.png">

Hbase 工作原理：

<img src="https://www.ibm.com/developerworks/cn/analytics/library/ba-cn-bigdata-hbase/image003.png">

## Mongodb架构 ##

<img src="https://docs.mongodb.com/v3.4/_images/sharded-cluster-production-architecture.bakedsvg.svg" >

### Minimum number of nodes to create a production-ready cluster ###

**Mongodb**： 3 nodes: Primary and secondary nodes deployed in a MongoDB replica set

**HBase** ： 10 nodes: Primary and secondary HMasters, RegionServers, active and standby NameNodes, HDFS quorum journal manager, DataNodes and Zookeeper ensemble

### Query processing ###

> When there are multiple selection criteria in a query, MongoDb attempts to use one single best index to select a candidate set and then sequentially iterate through them to evaluate other criteria.

如果查询条件组合没有能直接使用的索引，Mongodb也会自己去选择一个存在的，能用的索引对查询进行 优化。

> When there are multiple indexes available for a collection. When handling a query the first time, MongoDb will create multiple execution plans (one for each available index) and let them take turns (within certain number of ticks) to execute until the fastest plan finishes. The result of the fastest executor will be returned and the system remembers the corresponding index used by the fastest executor. Subsequent query will use the remembered index until certain number of updates has happened in the collection, then the system repeats the process to figure out what is the best index at that time.

对上面一段的解释，如何选择出“best index”来优化查询呢？先大概估计下哪几个可以用，然后生成几个候选的plan，在用户查询过程中执行plan，统计下时间，就算出来了。看上去是不怎么高级，没法直接分析出“best index”还要统计分析，原因是因为这“best index”是与文档的数目和结构非常相关的，例如以前使用的一个best索引体积突然膨胀很多，效率可能就会下降，这个时候，就不是best的index了。所以才有“Subsequent query will use the remembered index until certain number of updates has happened in the collection”。

> Since only one index will be used, it is important to look at the search or sorting criteria of the query and build additional composite index to match the query better. Maintaining an index is not without cost as index need to be updated when docs are created, deleted and updated, which incurs overhead to the update operations. To maintain an optimal balance, we need to periodically measure the effectiveness of having an index (e.g. the read/write ratio) and delete less efficient indexes.

并不是有多少查询条件，就建立多个个索引的，只要是实时维护的索引，都会有开销的，尤其是在插入的时候需要维护索引。所以，一些复杂的场景下，需要对索引进行周期性的评估，可能有100中查询条件组合，但是不应该有100个索引，而是需要精心的设计出有限少数个如5个索引，满足这100个查询条件的查询，这精心设计出来的5个索引当然也是根据数据的变化可能会变化的，所以说要“periodically measure the effectiveness of having an index (e.g. the read/write ratio) and delete less efficient indexes”，查看一个索引是否有真正的起到作用。

### Storage Model ###

> Written in C++, MongoDB uses a memory map file that directly map an on-disk data file to in-memory byte array where data access logic is implemented using pointer arithmetic. Each document collection is stored in one namespace file (which contains metadata information) as well as multiple extent data files (with an exponentially/doubling increasing size).

<img src="http://1.bp.blogspot.com/-g6bbn5_5dUY/T3oK8BowdcI/AAAAAAAAAmU/MgX_C74y89A/s400/p1.png">

上面几点关键的

- extend对应一个文档，大小是2次幂大小（有空间浪费）
- 为什么是2的次幂呢？一是对操作系统寻址快，4字节或8字节，二是可以预先批量分配空间，提高读写的吞吐量（自己猜的）
- 有预留的padding空间，用来处理一定范围内文档修改后增大，如果放不下会移动，也有空间浪费
- 索引是BTree？Mysql是B+树，B+树更适合文件系统，为什么Mongodb使用B树？

### Data update and Transaction ###
> In RDBMS, "Serializability" is a very fundamental concept about the net effect of concurrently executing work units is equivalent to as if these work units are arrange in some order of sequential execution (one at a time). Therefore, each client can treat as if the DB is exclusively available. The underlying implementation of DB server many use LOCKs or Multi-version to provide the isolation. However, this concept is not available in MongoDb (and many other NOSQL as well)

“可序列化”是并行执行的基础，如果操作不是可序列化的，有两个影响，一是不能通过网络传输，二是不能再服务器进行排队处理，不能排队就变成处理完一个客户端的请求在处理另外一个，这样，很有很多IO的等待，效率大大降低。而且排队处理还有另外一个作用，就是实现数据库的isolation特性，数据库可以根据isolation要求，对于客户端的请求进行合理的安排，从而实现了isolation。

在Mongodb中，没有隔离性，如果修改的时候基于以前的一个值，需要使用findAndModify方式，但这个方式也可能失败，执行完，可以在检查一次。

同样的，没有Transaction的特性，如果想事务地修改两个文档，也有一些其他的方式来实现，可以参考原文。

### Replication Model ###
> High availability is achieved in MongoDb via Replica Set, which provides data redundancy across multiple physical servers, including a single primary DB as well as multiple secondary DBs. For data consistency, all modifications (insert / update / deletes) request go to the primary DB where modification is made and asynchronously replicated to the other secondary DBs.

<img src="http://upload-images.jianshu.io/upload_images/6921295-8c04a5e509bf9a15.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/700">

有个Replica Set的概念，与fastdfs的Volume概念很像，只是多了个概念，方式与其他的一样，主从实现，一个组里面有个主的多个从的，写只写入主的，其他的从的异步同步。

> For data modification request, the client will send the request to the primary DB, by default the request will be returned once written to the primary, an optional parameter can be specified to indicate a certain number of secondaries need to receive the modification before return so the client can ensure the majority portion of members have got the request. Of course there is a tradeoff between latency and reliability.

这参数，很多都有，但是默认的是只写入一份就返回。

> For query request, by default the client will contact the primary which has the latest updated data. Optionally, the client can specify its willingness to read from any secondaries, and tolerate that the returned data may be outdated. This provide an opportunity to load balance the read request across all secondaries. Notice that in this case, a subsequent read following a write may not seen the update.

由于Mongodb的secondary 是异步的，如果在更新后立刻查询，可能查询不到刚刚更新或插入的值，所以默认的情况下，client是直接连接primary的。

> For read-mostly application, reading from any secondaries can be a big performance improvement. To select the fastest secondary DB member to issue query, the client driver periodically pings the members and will favor issuing the query to the one with lowest latency. Notice that read request is issued to only one node, there is no quorum read or read from multiple nodes in MongoDb.

提升性能，不是分片越多越好，大多数情况下，副本越多，提高的效率要比分片多高很多，在ES中，也是如此。如果使用原始的客户端，会与前面提到的选择“best index”一样，客户端会选择一个best的node。还有，在Mongodb中的client不会从多个相同数据的nodes中同时读取然后合并。Hbase会么？

> The main purpose of Replica set is to provide data redundancy and also load balance read-request. It doesn't provide load balancing for write-request since all modification still has to go to the single master.

在一个Replica Set中，没有写的负载均衡。目前为止，知道的不管是分布式的文件系统还是数据库都没有这特性，因为如果能向同一个中写，那同步的时候就太乱了。顶多像Kafka中的，可以把partition多搞出几个来，对应Mongodb，如果写是瓶颈，可以多个几个shard。

> Another benefit of replica set is that members can be taken offline on an rotation basis to perform expensive operation such as compaction, indexing or backup, without impacting online clients using the alive members.

可以利用备份服务器轮转地做离线整理（compact）、建立索引、备份，而不需要阻塞在线服务器，但是还是需要在线的主从的切换，应该支持吧。

> There is also a special secondary DB called slave delay, which guarantee the data is propagated with a certain time lag with the master. This is used mainly to recover data after accidental deletion of data.

用于进行误删除恢复数据用的。

### Sharding Model ###

> To load balance write-request, we can use MongoDb shards. In the sharding setup, a collection can be partitioned (by a partition key) into chunks (which is a key range) and have chunks distributed across multiple shards (each shard will be a replica set). MongoDb sharding effectively provide an unlimited size for data collection, which is important for any big data scenario.

<img src="http://upload-images.jianshu.io/upload_images/6921295-21bca3368451430e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/700" >



### 参考： ###
[MongoDb Architecture](http://upload-images.jianshu.io/upload_images/6921295-a064623b5acec1ca.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/700)  

## 运维 ##
部署上，Mongodb相对容易，主从方式，Hbase一般在HDFS之上，并且依赖Zookeeper，区分HMaster节点与Region节点，相对复杂。

可视化管理工具上，两者都有很多种，具体细节不详。

## 功能 ##

### 二级索引 ###
二级索引Mongodb支持比较好，而且可以后期根据业务动态的增加索引。

Hbase的查询都是通过RowKey，或者全表扫描再结合过滤器筛选出目标数据，但全表扫描，肯定很慢了。目前，对于Hbase实现二级索引，有多种方式：

- 如果要效率高要把多条件组合查询的字段都拼接在RowKey，一是如果条件多，都放在RowKey会太长效率低，二是如果有新增的业务，新增索引比较麻烦，要改RowKey需要重新rebulid数据库。对于海量的数据，RowKey如果太长，加入很多没有用的字段，体积增长会非常大，而且加重了IO的负担，弱化了Hbase快速扫描的优势。
- 使用ES+Hbase的方式，京东的评价系统是这么搞的。比较常见，但是有一定的开发和运维的工作。
- Hbase中加入索引列族的方式，没有“索引列族”的概念，只是一种变相实现索引的方式。实现方式可以参考
:[HBase二级索引的设计(案例讲解)](http://www.cnblogs.com/MOBIN/p/5579088.html)
- 其他开源的实现方案，如华为的[hindex](https://github.com/Huawei-Hadoop/hindex),已有开源的Phoenix支持类似功能，已经很长时间不维护了。
- 如上，Phoenix，参考官网[Secondary Indexing](https://phoenix.apache.org/secondary_indexing.html),解决方案思路差不多，记录索引表的key和原表的rowkey的对应关系。

此外，Mongodb中的二级索引也有很多种的类型，如地理空间索引、Hash索引，而且这些索引还有很多高级的特性如Sparse Indexes、TTL Indexes。

### 地理空间索引 ###
地理空间索引，Mongodb支持简单的，Hbase不支持。地理空间索引的支持Mongodb也只是简单的，但能满足很多场景下的需求。Mongodb中，地理空间索引相关的有2dsphere Indexes、2d Indexes、geoHaystack Indexes。

### 文本搜索 ###
新版的Mongodb已经支持了，通过Text Indexes的方式实现。Hbase中的Rowkey貌似支持一点，但是只是匹配，而Mongodb的目标是一个内置的搜索引擎。

### 查询支持 ###
Mongodb支持正则查找，范围查找，支持skip和limit等等，而hbase只支持三种查找：通过单个row key访问，通过row key的range，全表扫描。

### sharding 策略 ###
Mongodb支持两种策略：Hashed Sharding和Ranged Sharding。Hbase的shard策略应该只是Ranged Sharding，每个Region的的rowkey是连续的，利于scan。

### 聚合 ###
Mongodb对这支持的很好，而且有比较高级的Aggregation pipeline。HBase这方面不支持，但可以用MapReduce变相实现。

## 性能 ##


## 总结 ##
直接引用中的原文：
> MongoDB’s design philosophy blends key concepts from relational technologies with the benefits of emerging NoSQL databases. While HBase is highly scalable and performant for a subset of use cases, MongoDB can be used across a broader range of applications. The latter’s intuitive data model, rich query framework, native drivers, and lower operational overhead will often enable users to ship new applications faster and more easily than with HBase.

在超大规模的读写性能上，Mongodb并不如Hbase，Mongodb的最大特点，对开发人员友好，使用方便，加上Schemaless的特性开发速度快。360曾经用Mongodb在1天内就发布了一个产品。而且Mongodb的功能丰富，数据模型灵活，支持各种查询方式，基于Mongodb开发简单且速度快。

## 参考 ##
[The MongoDB 3.6 Manual](https://docs.mongodb.com/manual/)  
[MongoDB and HBase Compared](https://www.mongodb.com/compare/mongodb-hbase)  
[DB-Engines Ranking](https://db-engines.com/en/ranking)  
[An In-Depth Look at the HBase Architecture](https://mapr.com/blog/in-depth-look-hbase-architecture/)  
[Top 5 Considerations for Selecting Non-Relational Databases](https://www.ascent.tech/wp-content/uploads/documents/mongodb/10gen-top-5-nosql-considerations-june-2016.pdf)  
[别再用MongoDB了！](http://www.infoq.com/cn/news/2015/07/never-ever-mongodb)  
[每天200亿次查询 – MongoDB在奇虎360](http://www.mongoing.com/archives/715)  
[MongoDB Sharded cluster架构原理](http://www.mongoing.com/archives/2782)  
[MongoDB构架图分享](http://blog.nosqlfan.com/html/3887.html)  
[How Sharding Works](https://medium.com/@jeeyoungk/how-sharding-works-b4dec46b3f6)