---
layout: post
title: ES（五）Finite State Transducers
category: 存储系统
tags: es
---

在项目中，有大量的结构化的数据，大概10亿条左右，用Hbase，二级索引太麻烦，而且查询条件的组合非常多，大概有10个字段，任意几个字段都能自由组合，如果自己维护很多二级索引，会非常麻烦。然后，项目中就采用了ES，ES每列都自带倒排索引，但写查询语句的时候要注意term过滤的先后次序，一般区别度最大的字段在前面，尽快过滤掉更多的数据。

问题来了，从数据库的角度看，这ES虽然每个列都有倒排索引，但也是单一列上的索引，如果一个复杂的查询，涉及到假设7个字段，而且这个查询非常频繁，即使语句优化的再好，难道比有多字段的联合索引的数据库更快么？比如在7个字段上建立了联合索引的MongoDB或者Mysql。没有找到理论的依据，但是从直观判断上，如果不考虑内存问题，肯定时不如有联合索引的MongoDB来的快啊。

以上比较，其实有点牵强，ES擅长的是全文索引，只拿精确的匹配来比较其实并不公平。

那不涉及全文索引，在有大量集群节点环境下，数据量较大的情况下，这种精确的查询命中条数较多的情况下，ES的性能是可能超过Mysql或Mongodb的。这里说了很多的限制条件，实际情况中，可能并不是需要所有条件都要。下面简单分析下。

先看下，在有索引，返回多条数据的情况。Mongodb都干了啥，首先要知道，这种二级索引，如果shards是按照hash进行的，那么按照二级索引查询，是不知道在哪个shard的，是需要广播到所有的shard上的，然后所有的查询返回到汇总的处理节点，然后排序后返回给客户端。

其实ES也非常像，只不过没有专门的二级索引，只不过是一个Node上有多个shard，也是需要汇总结果，只不过没有二级索引，需要把各个条件的聚合的结果进行求交集操作。

像Mysql、MongoDB、ES这种，其实都是数据存储系统，要想速度快，除了分片，加副本等方式，最重要的就是尽量少的IO，尽量少的磁盘IO，B+Tree也是为了适合磁盘检索而设计的。较少IO的一个方式就是尽量少的表扫描，最好有索引，直接找到数据在磁盘上的位置，直接访问，所以，对于MySQL和MongoDB，合理的索引是非常关键的。但这又有一个问题，索引是一颗树，而且在数据量大的情况下，是一颗大树。

典型的B+Tree，理想情况下，树高为3，根节点常驻内存，理想情况，需要两次IO，能定位出在磁盘上的大概位置。但是，以上都是在数据规模相对不大的情况下，如果数据量特别大，要么，树特别高，要么一个节点中存放的数据特别多。树高了，IO次数就多了，节点中数据多了，从前到后的遍历节点中的数据也是需要花时间的，特别是在高并发的情况下，影响应该很明显。而且一个非常重要的是，数据量特别多，想提高效率，那么索引树要尽量的常驻在内存，但是内存这个时候，很可能就是个问题，频繁的读取索引树，也增加了IO的次数。

那么ES是如何处理的呢？通过上面的分析，可以看出，索引树变大，无法常驻内存，会大大降低效率。而ES，使用了这FST结构，这结构如下：

<img src="http://2.bp.blogspot.com/_4pUbN9gxhUI/TPk21wErb9I/AAAAAAAAAFM/dhPcsyo3KV4/s400/FSTExample.png">

引用lucene作者文中的一句话：

> Essentially, an FST is a SortedMap<ByteSequence,SomeOutput>, if the arcs are in sorted order. With the right representation, it requires far less RAM than other SortedMap implementations, but has a higher CPU cost during lookup. The low memory footprint is vital for Lucene since an index can easily have many millions (sometimes, billions!) of unique terms.

这东西看上去像前缀索引，很多的词，都会有公共的前缀，尤其是字母型的语言如英语，尤其是海量的数据的时候，如果使用这FST的结构，可以大大减少内存，然后这结构（相当于以前的B树索引）就能常驻内存，当然是有代价的，就是CPU使用率变高了，但是一般的数据检索系统，CPU一般是有剩余的。从机器的整体使用上看，这样能够发挥一个机器更多的综合性能，而不是IO满了，CPU还在空闲的状态。

这种拿CPU获取内存的思路，还是第一次用到，可能也是lucene的一个设计理念，与ceph很像，尽量可能利用机器所有的硬件。

[Using Finite State Transducers in Lucene](http://blog.mikemccandless.com/2010/12/using-finite-state-transducers-in.html)  
[Lucene学习总结之三：Lucene的索引文件格式 (1)](http://forfuture1978.iteye.com/blog/546824)   
[Lucene's RAM usage for searching](http://blog.mikemccandless.com/2010/07/lucenes-ram-usage-for-searching.html)  
[Apache Lucene - Index File Formats](http://lucene.apache.org/core/2_9_4/fileformats.html)    
[Choosing a fast unique identifier (UUID) for Lucene](http://blog.mikemccandless.com/2014/05/choosing-fast-unique-identifier-uuid.html)