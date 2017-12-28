---
layout: post
title: Kafka（二）场景分析
category: 消息传递
tags: Kafka 消息队列
---

## 使用场景 ##
- Messaging 当消息队列用
- Website Activity Tracking 用户行为记录
- Metrics监控用，java中也有meterics工具
- Log Aggregation日志聚合，作为一个日志中心使用
- Stream Processing 流处理，与spark streaming类似
- Event Sourcing 有点抽象，事件流驱动
- Commit Log 功能类似数据库的日志，可以顺序恢复操作或消息


## 人脸项目使用 ##
- 持久化 : cgs->缓冲服务器(调用特征提取，生成id)->kafka(特征) + Hbase(大图小图)  
- 告警流程 : 缓冲服务器->kafka->告警判断(拉取比对库对比，spark stream)->告警放入kafka->应用服务器拉取
- 订阅发布流程 : 缓冲服务器订阅所有的事件消息->web登录后查询并缓存所有缓冲服务器地址->web订阅消息给缓冲服务器->缓冲服务器根据订阅策略分发消息到web
  
如何发布给用户，每个用户一个topic或流处理分析？ --No  
所在的流程节点，在告警分析后？如果在后面，那么发布有延迟；并行处理的？流分开处理了额么，如果分开同步处理，那么可以有单独的服务器再次拉取，费流量。--并行处理的

订阅分发考虑主要有两个因素，用户数量，事件更新的频率

解决方式有如下：
- 所有卡口最新的事件缓存到队列，按照时间缓存，比如前端5秒钟拉一次数据，那么简单的方式，事件更新不频繁，缓存都放在内存或Redis中，查询如果频繁，放到内存，如果不怎么频繁Redis。这能解决大多数的问题，因为缓存的只是消息体，图片需要去自己拉取。每个消息1k，每秒有1k的消息，缓存5秒，才5M的体积。
- 如果消息比较多，这是一般的场景，可以使用spark stream根据订阅分发策略进行处理，如果订阅用户不多，那么可以放入kafka的topic中，订阅者一般是web端，这可能要有限流策略，有个消息的丢弃策略，否则更新太快，前端也拉取不了，这偏应用，web端自己做。但是每个用户一个topic的方式，对于kafka也是个问题，如果topic不及时清理，或者突然用户增多，虽然kafka能支持10000个topic，但是每个topic都有对应的文件，到时候IO对系统会产生比较大的影响。

为了达到顺序的目的，msk的key是卡口设备的id，这样就能保证一个卡口下的事件到达同一个partition这样，在这个partition内就是有顺序的。为了保证以后提高吞吐量时候，partition的num导致的瓶颈，一般partition会有“预partition”的操作。

## 实战项目性能分析 ##
人脸项目使用经验，kafka远没有达到性能使用的瓶颈。

官网推荐的测试一个benchmark：

> Single producer thread, no replication  
> 821,557 records/sec  
> (78.3 MB/sec)  

消息大小是100byte大小，有80w每秒的写入速度。这里，测试只用一个小的100byte的消息是应为更能测试出records/sec，如果每个消息体积越大，在MB/sec上的吞吐量会有更好的数据。

读取上，一个简单的测试结果，同样是100byte大小：
> Single Consumer  
> 940,521 records/sec  
> (89.7 MB/sec)  

机器配置如下：

> Intel Xeon 2.5 GHz processor with six cores  
> Six 7200 RPM SATA drives  
> 32GB of RAM  
> 1Gb Ethernet

非常普通的机器，吞吐量写入有80w，读取有90w，远远满足于现在项目需求。项目中，写入大概数量级在1k左右，读取不会超过1w。

从IO方面上分析，如果一条消息1k，网卡90MB/s的速度，单千兆网卡有9w的吞吐量，也是没什么问题的。

## 实战平台使用流程 ##

### 事件持久化 ###
接入服务器->图片写入云存储->kafka(过车事件topic)->数据清洗->告警分析->入库

## 订阅发布 ##
应用服务器订阅所有过车事件->web使用websocket连接服务器->应用服务器实时对过车数据分析推送到各个web

## Demo ##
默认情况下，Kafka根据传递消息的key来进行分区的分配，即hash(key) % ，如果消息没有key，Kafka几乎就是随机找一个分区发送无key的消息

如果你的分区数是N，那么最好线程数也保持为N，这样通常能够达到最大的吞吐量。

## 资源消耗 ##
IO密集型，引用[某互联网大厂kafka最佳实践](https://www.jianshu.com/p/8689901720fd)中的一段：

> 不建议为kafka分配超过5g的heap，因为会消耗28-30g的文件系统缓存，而是考虑为kafka的读写预留充足的buffer。Buffer大小的快速计算方法是平均磁盘写入数量的30倍。推荐使用64GB及以上内存的服务器，低于32GB内存的机器可能会适得其反，导致不断依赖堆积机器来应付需求增长。（我们在生产环境使用的机器是64G内存的，但也看到LinkedIn用了大量28G内存的服务器。）

10w吞吐量，单台应该是不够了，按照上述描述，每个消息1k，读写一共只有10个，需要5+30+1k*10w*10 = 36G

## topic 应该有多少 partition  ##

参考：[How to choose the number of topics/partitions in a Kafka cluster?](https://www.confluent.io/blog/how-to-choose-the-number-of-topicspartitions-in-a-kafka-cluster/)

> On the consumer side, Kafka always gives a single partition’s data to one consumer thread. Thus, the degree of parallelism in the consumer (within a consumer group) is bounded by the number of partitions being consumed. 

这里有点，一个partition同一个时刻只给一个consumer进行消费，原因其实不复杂，最大利用磁盘顺序读取的速度么。


## 参考 ##

[Kafka设计解析（一）](http://www.jasongj.com/2015/03/10/KafkaColumn1/)  
[某互联网大厂kafka最佳实践](https://www.jianshu.com/p/8689901720fd)  
[Kafka下的生产消费者模式与订阅发布模式](http://blog.csdn.net/zwgdft/article/details/54633105)  
[How to choose the number of topics/partitions in a Kafka cluster?](https://www.confluent.io/blog/how-to-choose-the-number-of-topicspartitions-in-a-kafka-cluster/)  
[Benchmarking Apache Kafka: 2 Million Writes Per Second (On Three Cheap Machines)](https://engineering.linkedin.com/kafka/benchmarking-apache-kafka-2-million-writes-second-three-cheap-machines)