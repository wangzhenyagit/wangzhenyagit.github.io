---
layout: post
title: sofa registry介绍
category: sofa
tags: sofa
---

## 介绍
注册中心对应的也有sofaRegistry，registry本来就是注册中心的意思，这次取名字有些过于朴实。主要特性如下：

- 支持服务发布与服务订阅
- 支持服务变更时的主动推送
- 丰富的 REST 接口
- 采用分层架构及数据分片，支持海量连接及海量数据
- 支持多副本备份，保证数据高可用
- 基于 SOFABolt 通信框架，服务上下线秒级通知
- AP 架构，保证网络分区下的可用性

特点上与Eureka类似AP，只是单纯的注册中心。

## 架构

文档中的架构图：
![](https://gw.alipayobjects.com/zos/basement_prod/a9b69b25-836f-4bbe-a32c-ec6148084f93.svg)

架构图中，有单独的Session Cluster，这个Session Cluster名字就有些吓人，只是为了与服务保持个Session就搞个单独的集群，更像现代的软件架构。其余的两个部分Data Cluster和Meta Cluster是为了保存数据，看到这两个结构想起Hbase，但Hbase的存储数据量是亿级别的，这个单独的服务注册中心，也是这样的结构而存储的数据和元数据只是一些应用服务的信息，集群中服务的信息，把大数据的存储架构用来存储这些数据，估计用很多年都不是问题。

## 其他的注册中心
### Consul 
> Consul is a service mesh solution providing a full featured control plane with service discovery, configuration, and segmentation functionality. 

- 是个多面手，不仅仅是个注册中心，可以用来做数据一致性的组件，配置管理中心“Consul provides a super set of features, including richer health checking, key/value store, and multi-datacenter awareness.”
- 强一致性，“Consul provides a strong consistency guarantee, since servers replicate state using the Raft protocol. ”“The strongly consistent nature of Consul means it can be used as a locking service for leader elections and cluster coordination.”
- 支持跨数据中心的数据同步，Gossip协议，和Redis Cluster的协议一样
- 支持多种健康状态检查

### Nacos
Nacos与Consul很像，健康检查、多数据中心、可做配置中心，比Consuls多了雪崩保护功能，其他的区别暂不清楚。

### ZK、etcd
- Zk、etcd是CP的，并不合适，设计之初这两也不是为了做为注册中心
- 健康检查很单一

### Eureka
- AP的，很合适
- 健康检查支持，需要自己配置
- 多数据中心不支持

很少讨论sofaRegistry的文章，在github中的一个Issues问题：请教和nacos的对比,有什么特色功能？

> SOFARegistry 和 Nacos 最主要的不同是：Nacos 把配置管理和服务发现的功能集成在一起；SOFARegistry 专注于服务发现和服务注册能力，对于配置管理部分会有另外的配置中心产品进行独立承载职责，这样的划分形式希望职责分明，每种能力都做到最强。

## 参考  
[服务发现比较:Consul vs Zookeeper vs Etcd vs Eureka](https://developer.aliyun.com/article/759139)  
[Consul vs. Eureka](https://www.consul.io/intro/vs/eureka)




