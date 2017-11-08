---
layout: post
title: Hbase （一）概述
category: 存储系统
tags: Hbase
keywords: Hbase
---

## Hbase是什么
从本质上看，其实Hbase是个存储系统，已经有了HDFS为什么还需要Hbase呢，而且Hbase是基于HDFS的。如果只从存储的角度上看，HDFS对于实时的随机存储支持较差，而且对于元数据节点的压力，HDFS支持海量的小文件并不好。为了解决上述问题，可以在HDFS上对比较大的块文件增加索引，快文件内部是很多的小文件，这样检索的时候先读到大文件的索引的部分，在利用索引快速读取到要的小文件，能解决部分问题，然后，就有了HBase。同样的，Google的GFS也有此问题，后来就有了BigTable。所以，只从存储上看，HDFS解决了可靠性与扩展性的问题，而Hbase解决了随机存储与海量的小文件存储问题。

既然HDFS有这样的问题，那么问题来了，HDFS会不会把随机存储和海量小文件都支持好呢？额，这个不说，如果要支持结构上可能会有大的改动，但是如果不支持作为一个存储系统，就显得不是那么通用，毕竟ceph已经基本解决了上面的问题。但是有一点可以肯定的是，即使解决了上面两个问题，HDFS也不会支持上所有Hbase的功能，关注的点不一样，解决事情的场景也不一样。

HBase 并不适合所有问题，其设计目标并不是替代 RDBMS，而是对 RDBMS的一个重要补充。简单理解，放开了范式的束缚，提高了写入与查找速度。

## 适用场景
- 需要确信业务上可以不依赖 RDBMS 的额外特性，例如，列数据类型, 二级索引，SQL 查询语言等。不过二级索引、SQL查询在Hbase上现在也能支持了。
- HDFS使用最好多余5个数据节点，(根据如 HDFS 块复制具有缺省值 3), 还要加上一个 NameNode。但也不是绝对的，随着硬件的发展，软件的配置也在改变。
- 大规模，上亿条可适用。并不是说上亿行就非要Hbase了，Mysql在亿级处理下很多场景也是可以的，对于偏重OLAP的，很多时候是全表扫描的，如果对延迟不敏感Mysql也能用。

## 模块构成

<img src="https://www.ibm.com/developerworks/cn/analytics/library/ba-cn-bigdata-hbase/image002.png" />

Hbase有单独的master，而kafka没有master，所有master的功能都是在zk上实现的。与Kafka的相比，Hbase的“管理”工作还是比较多的。
- HBase Master 用于协调多个 Region Server，侦测各个 Region Server 之间的状态，并平衡 Region Server 之间的负载
- HBase Master 负责分配 Region 给 Region Server。
- The Master controls critical functions such as RegionServer failover and completing region splits. master处理全局的关键操作，region Server的failover和regionsplites。如果没有master可以使用zk选举leader来做，但是还是有确定的Master来做更简单，而不是把Region升级为leader。

## 工作原理

<img src="https://www.ibm.com/developerworks/cn/analytics/library/ba-cn-bigdata-hbase/image003.png" />

**路由**  
当一个 Client 需要访问 HBase 集群时，Client 需要先和 Zookeeper 来通信，然后才会找到对应的 Region Server。这和kafka查broker和topic一样。

首先client会与ZK集群联系，从ZK那里询问哪一个节点持有-ROOT-这个region？ 然后，client向该RS询问，.META.表中包含target rowkey的region处于哪一个节点上？（以上这些查询的结果都将被client缓存起来）。Redis cluster也是客户端缓存路由信息

最后，client访问target server，与RS通讯，进行真正的数据查询。

**WAL**  
HBase 中的 HLog 机制是 WAL 的一种实现，而WAL（一般翻译为预写日志）是事务机制中常见的一致性的实现方式。每个 Region Server中都会有一个HLog的实例，Region Server 会将更新操作（如 Put，Delete）先记录到 WAL（也就是 HLog）中，然后将其写入到 Store 的 MemStore，最终 MemStore会将数据写入到持久化的 HFile 中（MemStore到达配置的内存阀值）。这样就保证了 HBase 的写的可靠性。单个节点收到数据后快速持久化，并不是真正写到region里面，而是先写到region Server的“快速持久化”的“缓存”

当client向HBase写入一条数据时，首先会将数据写入到WAL（write-aheadlog）中。（WAL的作用是在RS崩溃后，还能恢复尚未被持久化的数据。）当数据被写入到WAL后，就会被放入到MemStore中。同时，会检查MemStore是否已满（缓冲区大小由hbase.hregion.memstore.flush.size配置，默认值是64MB），如果已满则将将其flush到磁盘上。该flush请求由RS中一个单独的线程负责，该线程会将数据写入到HDFS中的一个新的HFile文件中。像写的2级缓存，第一级缓存主要是快速持久化，加快返回速度，第二级是批量写，减少写入HDFS的次数，增加效率。

通过memstore可以实现批量写磁盘，通过多个memstore轮流工作，可以在写磁盘时候不阻塞log写入memstore。与我司存储服务器很像，看来很多问题有共同的解决方式。

虽然有预写日志在到内存再到磁盘的过程，但是不会有读的延迟，会把磁盘上的文件和内存中合并，没有用到wal中的内容，因为数据几乎是同时到达wal和memstore中的，wal更多的是为了快速持久化处理断电异常等。这里与es处理方式也很像，es在插入文档后，为了防止数据丢失也有日志记录，然后会从内存先写到磁盘的缓存（软缓存，其实也是内存），周期是1s，检索的时候，会读取缓存中的文件和磁盘上的文件，这样，只会有近似实时（1s的延迟）的滞后。

**合并**  
Hbase中存在两个合并操作，从memstore写到磁盘就是一个hfile，这样文件太多，不好管理。需要把多个小文件写入一个大文件，这过程较minor compaction。
此外还有更重要的major合并，列族聚集顺序重写删除无效数据。

在ES中也有文件合并的操作，而且在ES中是把已经提交的段与未提的段（在文件系统缓存中）进行合并，这样还能够减少文件缓存到磁盘持久化的过程。

## 存储的数据结构

<img src="https://www.ibm.com/developerworks/cn/analytics/library/ba-cn-bigdata-hbase/image001.png">

>稀疏的、分布的多维的 Map

### 稀疏的
如pic中，不是每个“格子”都填满，逻辑上是个稀疏矩阵，但是物理存储上并没有浪费，完整的应该逻辑上矩阵稀疏的

### 分布的 
每个CF都存在不同的HFile中，也就是查一个行key会查找分布的多个HFile，列族分布存储。

### 多维的
具体的应该是四维，而不是关系数据库的二维，四个维度为行key，列族，列name，时间（版本）

### map
列族上存储的是map的形式，三个键组合对应一个值，行key，列name，版本对应一个存储的值

## 结构带来的优势？

### 性能
**缓存**  
NoSQL数据库都具有非常高的读写性能，尤其在大数据量下，同样表现优秀。这得益于它的无关系性，数据库的结构简单。一般MySQL使用Query Cache，每次表的更新Cache就失效，是一种大粒度的Cache，在针对web2.0的交互频繁的应用，Cache性能不高。而NoSQL的Cache是记录级的，是一种细粒度的Cache，所以NoSQL在这个层面上来说就要性能高很多了。

**key-value**  
不仅扩容简单，基于键值对的，可以想象成表中的主键和值的对应关系，而且不需要经过SQL层的解析，所以性能非常高。

**并发操作**  
Hbase对于行的操作是原子的，这样的锁相对于关心型的数据库，粒度会很少，在并发的写的时候会很快。

而且版本的存在，不仅仅是为了使用上存储不同版本的值，处理一些特殊的业务需求。有了版本的存在，在更新的时候，甚至都不用管以前的记录，直接追加写入，然后经过major合并整理，或者读取的时候根据版本过滤能得到最新的值，而且对于并发的写也是安全的，完全可以多个客户端同时的写，看上去同时写，实际上只是各自记录了流水账，你们写的都有用，都会以各种版本的形式保存下来。想着吞吐量上就大。

**压缩**  
列式存储有利于压缩，前缀压缩算法特别适合这张场景，相同类型的字段在一起压缩效率很高。这里可能有同学有疑问了，压缩不是耗费cpu么，不是会更慢么？这里有个度的问题，实际中的场景是，需要读取大量的Hbase文件，而磁盘的速度只有几十兆，如果不压缩这些数据，尽管是分布式，但瓶颈会最终在磁盘IO上。即使压缩了大多数场景下，磁盘IO也是主要瓶颈。需要压缩的列外一个原因就是，Hbase是列式存储，还是key-value的形式，这就导致数据中很多rowkey的重复冗余字段。

**使用假设**  
> hbase假设不是每次需要所有的列

而且设计上有列族的概念，相关的列在一起，在一个列族，在一个hfile上，而且压缩效率高。这样的话，如果每次都是获取“部分相关的列”（列族），那么读特定的hfile而不是所有的列，速度又会快很多。

**负载均衡**  
另外，hbase也有自动的负载均衡机制，会把负载较高的region移动到不繁忙的服务器节点上，通过region的移动让各个服务器达到负载均衡。额，这点与HDFS有些类似，会移动IO繁忙的数据块在不同的服务器上移动。

PS:hdfs上也存在多个副本，有没有利用多个副本的读负载均衡？有的话Hdfs也会受益。

### 扩展性
因为基于键值对，数据之间没有耦合性，所以非常容易水平扩展。

region是按照行键分的，当一个region过大，region会自动分裂，对上层是透明的，而且没有es坑人的shard分片变化要重建集群的麻烦。

## 参考
《HBase权威指南》

[Hbase深入浅出](https://www.ibm.com/developerworks/cn/analytics/library/ba-cn-bigdata-hbase/index.html)

[HBase压缩机制介绍](http://forum.huawei.com/enterprise/thread-327123-1-1.html)
