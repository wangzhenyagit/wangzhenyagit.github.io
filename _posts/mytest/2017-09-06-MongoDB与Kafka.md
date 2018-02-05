---
layout: post
title: MongoDB与Kafka性能测试
category: 性能测试
tags: MongoDB
---

### 测试目标 ###
测试功能，从kafka拉取JSON序列化后的结果，反序列化，并插入到Mongodb。  
测试环境为，Intel(R) Xeon(R) CPU E5-2650 v4 @ 2.20GHz，128G内存

测试代码路径：

### JSON反序列化的效率 ###

单线程运行，在i7-6700，单颗CPU3.4GHz,结果是：
```
spend : 3053ms, 32.0w per sencond
spend : 196ms, 510.0w per sencond

```

单线程运行，在Intel(R) Xeon(R) CPU E5-2650 v4 @ 2.20GHz,结果是：

```
spend : 5497ms, 18.0w per sencond
spend : 288ms, 347.0w per sencond

```

#### 结论 ####
单线程下，速度与单核心CPU的主频相关很大。  
JSON对象的序列化，与它的元素的多少关系很大。
数量级上是百万级别到十万级别。

### MongoDB插入速度 ###
插入速度测试，由于没有千兆网络，直接在同一个机器上测试。

单线程下:
速度稳定后：
```
test count : 492, spend : 297651ms, 16000.0per sencond
```

磁盘IO还不是瓶颈，sdj上是4块7.2k的RAID10：
```
Device: rkB/s    wkB/s avgrq-sz avgqu-sz   await r_await w_await  svctm  %util
sdj     1.33  3693.17     9.58     0.05    0.07    5.00    0.07   0.04   3.40
```

util才3.4%，写速度也只有3M

使用mongostat查看
```
insert query update delete getmore command dirty used flushes vsize   res qrw arw net_in
  9979    *0     *0     *0       0     1|0  0.0% 6.2%       0 5.08G 3.74G 0|0 1|0  17.3m
 20016    *0     *0     *0       0     2|0  0.1% 6.3%       0 5.12G 3.78G 0|0 1|0  8.67m

```

单线插入的情况下，能够达到1.6w每秒的速度。但是磁盘IO，CPU等均没有达到瓶颈。MongoDB并没有把所有的资源给一个客户端。修改下批量的大小到100、1000、甚至10w一批，速度变化不是很大，100一批的时候也有1.4w的速度，因为不是在网络环境下，插入程序与MongoDB server在一台机器上，测试意义不大。

加了两个索引，但是看上去对插入速度没啥影响。

#### 结论 ####
单线程下，MongoDB的插入速度稍复杂对象，有1.6w左右，如果生产环境中只有单线程在写入，那么可能是一个瓶颈所在。

如果不是网络环境下，batch的大小对插入速度的影响并不大。

### kafka插入速度 ###
cpu信息与上面一样，使用一块磁盘7.2k的SAS盘，没有RAID，单线程情况下，对较长的字符串插入，长度834，速度有大概9w每秒的速度，默认的broker配置，top看了下，单线程下应该是CPU的问题，虽然后48个核心，但是对一个client分配的cpu资源是是有限的，top下的cpu使用率是100%多一些，应该是某一个线程达到了瓶颈。

IO看了下util没有满，然后加到两个线程，cpu使用率为200%，看看IO，util使用率经常性的100%，但不是一直繁忙，应该IO还不是问题。还可能是CPU的问题。两个线程大概有14w的吞吐量。

加到了4个线程，IO的速度一直维持在150M左右，util绝大部分是100%了，马上就是IO的瓶颈了。吞吐量是20w左右。

加到6个线程，与上面猜想一样，IO是瓶颈了，吞吐量没有在增长20w左右。

官网上的速度有

值换成短字符串，长度32, 6线程同时，IO不是问题，有大概100w的吞吐量。

