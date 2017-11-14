---
layout: post
title: Cassandra介绍
category: 存储系统
tags: Cassandra
---

在DB-engine上看到的排名，写此文时间，排名第七，而同样为wide column数据库的Hbase排名为15。这里比较下Hbase的异同。

### 功能 ###
在[db-engines](https://db-engines.com/en/system/Cassandra%3BHBase)上的对比，两者也很像。总结下，相同点：都是Wide-column，都是2008年发布，都有Sharding，访问控制，自动负载均衡等等，不同点，Consistency concepts上cassandra是灵活的，而Hbase是Immediate Consistency，从功能上看，几乎差不多。

### 结构 ###
Hbase依赖于Zk，并且有HMaster节点，理论上有点单故障，而Cassandra节点直接相等，节点之间用Gossip协议，无单点问题。但是这Gossip协议如果使用不好，在多个节点（比如上千个），好像有风暴的问题，此问题尚不确定。

存储上，Hbase设计根基与bigtable，以scan的方式为基础，rowkey连续的话，很可能会在一个region上，这方便与连续的读，Hbase这种方式适合scan，与MapReduce很合适，而且在扩展的时候，region分裂相对方便。而Cassandra设计根基于Dynamo，采用rowkey的hash方式，相同的rowkey很可能不在一个shard上。这也是Hbase与Cassandra本质的区别。

数据结构决定了两者的优势与使用场景，在有多节点情况下，读写的速度Cassandra比Hbase快。但是，一定场景下，如Mapreduce这种，并发连续读取，Hbase的速度是比Cassandra快的。另外一个亮点，设计上看，读写的速度是矛盾的，而Cassandra能通过参数配置，调节读写的吞吐量。

这种hash的方式也符合一般的负载均衡的方法。Cassandra的hash分片方式与Redis很像，虽然Cassandra比Redis多了一个维度，可以降一个维度，在大数据量上内存不大够用的时候，能替换Redis的功能。

其实两者的写入速度都很快，优化写入速度的方式与大多数一样，都是写内存和日志，然后内存批量到磁盘，然后再整理合并，ES、Hbase、Cassandra等等很多。

参考知乎
[cassandra与hbase的利弊分析？适用场景？社区环境？](https://www.zhihu.com/question/20152327/answer/95843437)的回答。总结的使用场景：
> 应用场景需要大量scan操作 或者需要经常配合MapReduce，而random access数据为辅助手段，那么HBase是你的绝佳选择。
> 
> 如果你需要高并发可调节读写，scan需求少，那么Cassandra则比HBase更合适。

### 场景 ###
HBase与Hadoop很方便，而Cassandra与Spark相配，目前由于Spark对于Hadoop占据了优势，从而使得Cassandra占据了很大优势。Cassandra+Spark vs HBase+Hadoop 上，C+S已经占据了明显的优势。

### 总结 ###
- Cassandra部署相对简单，没有master节点，不依赖zk；
- Cassandra数据一致性可以配置、读写可以配置；
- Cassandra适用于scan操作较少，读写吞吐量较大的场景；




