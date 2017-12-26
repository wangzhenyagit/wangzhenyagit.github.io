---
layout: post
title: ES（五）Finite State Transducers
category: 存储系统
tags: es
---

在项目中，有大量的结构化的数据，大概10亿条左右，用Hbase，二级索引太麻烦，而且查询条件的组合非常多，大概有10个字段，任意几个字段都能自由组合，如果自己维护很多二级索引，会非常麻烦。然后，项目中就采用了ES，ES每列都自带倒排索引，但写查询语句的时候要注意term过滤的先后次序，一般区别度最大的字段在前面，尽快过滤掉更多的数据。

问题来了，从数据库的角度看，这ES虽然每个列都有倒排索引，但也是单一列上的索引，如果一个复杂的查询，涉及到假设7个字段，而且这个查询非常频繁，即使语句优化的再好，难道比有多字段的联合索引的数据库更快么？比如在7个字段上建立了联合索引的MongoDB或者Mysql。没有找到理论的依据，但是从直观判断上，如果不考虑内存问题，肯定时不如有联合索引的MongoDB来的快啊。

以上比较，其实有点牵强，ES擅长的是全文索引，只拿精确的匹配来比较其实并不公平。

那不涉及全文索引，在有大量集群节点环境下，数据量较大的情况下，这种精确的查询命中条数较多的情况下，ES的性能是可能超过Mysql或Mongodb的。这里说了很多的限制条件，实际情况中，可能并不是需要所有条件都要。

先看下，在有索引，返回多条数据的情况。Mongodb都干了啥，首先要知道，这种二级索引，

[Using Finite State Transducers in Lucene](http://blog.mikemccandless.com/2010/12/using-finite-state-transducers-in.html)  
[Lucene学习总结之三：Lucene的索引文件格式 (1)](http://forfuture1978.iteye.com/blog/546824)   
[Lucene's RAM usage for searching](http://blog.mikemccandless.com/2010/07/lucenes-ram-usage-for-searching.html)  
[Apache Lucene - Index File Formats](http://lucene.apache.org/core/2_9_4/fileformats.html)    
[Choosing a fast unique identifier (UUID) for Lucene](http://blog.mikemccandless.com/2014/05/choosing-fast-unique-identifier-uuid.html)
[Lucene学习总结之三：Lucene的索引文件格式 (1)](http://forfuture1978.iteye.com/blog/546824)