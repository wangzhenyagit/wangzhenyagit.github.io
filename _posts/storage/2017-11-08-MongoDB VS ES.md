---
layout: post
title: MongoDB VS ES
category: 存储系统
tags: Mongodb Elasticsearch
---

参考[DB-engine](https://db-engines.com/en/system/Elasticsearch%3BMongoDB)上的对比，从功能上看，如果不算权重，Mongodb功能相对还是比ES要多一点，捡主要的分析下。

### 总体比较 ###
ES是Java开发，而Mongodb是C++开发。使用Java虽然先天跨平台，但有不少问题，比如经常有GC时间长，个人感觉，可能会成为制约ES发展和性能的一个问题。**相同点**：两者都不支持SQL，没有事务支持，而且都是schema-free，都有Sharding和Replication，都有二级索引，都支持简单的聚合。**Mongodb比ES多的功能**：Mongodb有访问控制，而ES的访问控制是开源维护的公司留着收钱的，像ngnix的监控功能；Mongodb有Mapreduce功能。

### 排名与活跃度 ###
参考[DB-engine](https://db-engines.com/en/ranking)此文的时间，2017年11月，Mongodb排名第5，前四名分别是Oracle、Mysql、SQL Server、PostgreSQL，是文档型数据库中的第一名，是将近第二名9倍的分值；而ES排名刚刚进入前十是搜索引擎排名中的第一名，是第二名solr的2倍分值；而且这两个开源的软件都是这两年突然火起来的，ES在16年初的时候，分值还不如solr，而不到两年，是solr的将近两倍。这个排名分值包括了Git hub活跃度，各种社区论坛的活跃度和各个公司使用的情况。

### Schemaless ###
ES的Schemaless是针对增加free，但是一旦map中的类型固定了，是没法在修改的。而Mongodb的Schemaless支持的更好，从Mongodb的支持的脚本就能看出来了JavaScript的。

Schemaless是非常有利于项目需求不定的时候，快速开发，快速修改。

但是，使用的时候需要注意两个问题，Schemaless灵活，但是很不会更容易出错？另外一个问题，ES中map类型确定就不可以修改，但是如果能够修改，从效率上看应该会损失很多，而且使用中也更容易出错了。

### Mongodb+ES如何使用 ###
ES支持插件，很多插件都是为了从主数据库导入到ES的，而从Mongodb导入的插件为Mongodb-river。可以参考：[How to use Elasticsearch with MongoDB?](https://stackoverflow.com/questions/23846971/how-to-use-elasticsearch-with-mongodb?rq=1)

### MongoDB + ES or only ES ###
在做过车数据存储的时候遇到的问题，需要支持模糊搜索，聚合，查询过滤。

针对场景进行分析，模糊搜索功能ES强项，所以方案中必须有ES了。虽然新版本的Mongodb也要支持上搜索的功能，但目前看还是出于试验阶段，而且lucene相对成熟，如果Mongodb不基于Lucene。

而聚合和查询功能，其实ES与Mongodb都是支持的，如果只从以上几点看，单独使用ES是可以的。

使用单独的ES，能够好的利用cpu和磁盘，数据不用在多个地方重复，cpu在不进行查询的时候可以给聚合统计使用。

如果聚合、查询、搜索的QPS非常高，那么也可以单独配置ES为搜索业务，Mongodb为存储查询聚合功能，功能上能够分离，维护上比较好。

另外，ES是基于Java的，而且在Github上很多的OOM问题，自己也遇到了OOM，所以从整个系统的稳定性上考虑Mongodb做存储、过滤和聚合，ES做搜索分工，两者独立，系统相对稳定。

性能上，需要根据场景进行特定的测试才行。

最后，在Github上和知乎上都有建议，如果ES的稳定性，并不是很好，**如果要使用ES做为唯一的数据存储，要慎重。**查了一圈，得到了一个这模模糊糊的，没有数据支持的结果。如果非要找数据，ES的github上有100多个pull request，而mongodb的只有26个。

### MongoDB ES 比 Mysql优势 ###
这里不做详细展开，目前来看Mysql的shard与replica机制使用起来是比ES和Mongodb麻烦的，而且在Schemaless上也不如Mongodb和ES，但是如果这些都不考虑呢？如果数据量不大，需求比较固定，目前看，还是Mysql比较合适的，稳定高效。如果是OLAP的，需要全表扫描的，Mongodb和ES，在全表扫描上可能都不如Mysql，如果有主键索引的地址是连续的，能够批量预读，而且能利用NUMA，速度杠杆的。如果是表全字段的扫描，也可能比Hbase快，Hbase不同字段在不同的地址，还要解压。所以，场景很重要。

## 参考 ##
[MongoDB + Elasticsearch or only Elasticsearch?](https://stackoverflow.com/questions/29538527/mongodb-elasticsearch-or-only-elasticsearch)

