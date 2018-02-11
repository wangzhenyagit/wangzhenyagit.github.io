---
layout: post
title: To Think List
category: 我的思考
tags: 总结
---

- OOM问题如何避免？
- mmap机制？https://stackoverflow.com/questions/19280412/mongo-server-status-resident-memory 
- Mongod的两种引擎 https://docs.mongodb.com/manual/core/wiredtiger/#storage-wiredtiger  https://docs.mongodb.com/manual/core/mmapv1/#storage-mmapv1  
- Json的最佳实践，fastJson的Git相关文章
- Mongo、HDFS、FastDfs、ES、是否需要raid技术
- 如何使用puppet进行部署
- Mysql的缓存机制有哪些?
- 插入数据的时候，ES返回成功的时候，已经持久化到磁盘了么？Mysql的高速插入，断电的话会有1s的数据丢失 https://dev.mysql.com/doc/refman/5.7/en/myisam-key-cache.html   
- java 的ThreadPoolExecutor状态转移？ http://www.jianshu.com/p/ade771d2c9c0
- LongAdder能代替AtomicLong?
- 一种架构https://discuss.elastic.co/t/performance-help/44025/2
- yarn 资源管理，如何管理，管理什么？
- 配置管理工具（ Puppet，Chef，Ansible）
- 慢日志机制
- 很多都是使用hash来计算一个资源应该去哪个shard访问，对应的，监控系统中资源为摄像头，如果是集群的状态，可以通过摄像头的id进行hash计算，然后分配哪几个shard接入哪几个设备。而一个服务器node上能够容纳多个shard。资源的均衡是根据shard为单位进行的。
根据shard为单位均衡，而不是根据具体设备来均衡，一是设备数量可能很多10w，如果按照10w来进行均衡，均衡工作就会很复杂，master节点负载会很大。
- [滚动重启机制](https://www.elastic.co/guide/cn/elasticsearch/guide/cn/_rolling_restarts.html#_rolling_restarts) 对于接入集群也可以参考
- 为什么需要时序数据库？ https://db-engines.com/en/blog_post/71 
- 并发编程的坑？ http://blog.csdn.net/solstice/article/details/5915355
- lock free的结构 https://stackoverflow.com/a/260277
- 并发好的结构的选择 http://www.drdobbs.com/parallel/choose-concurrency-friendly-data-structu/208801371?pgno=2


### To learn 
- 京东618大促网关承载十亿调用量背后的架构实践 http://mp.weixin.qq.com/s/6nYROabwFT1o9ppH2TbPhA  
- 人工智能在线特征系统中的数据存取技术 https://mp.weixin.qq.com/s/kPmvBCcS0O-jTgE_VAysug  
- 每秒处理10万高并发订单的乐视集团支付系统架构分享 http://www.zuidaima.com/blog/2852602576456704.htm  