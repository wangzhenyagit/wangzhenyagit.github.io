---
layout: post
title: 负载均衡技术
category: 我的技术
tags: 负载均衡
keywords: 负载均衡 nginx Haproxy
---

# 概述
## 使用场景
- 应用于高访问量的业务
- 横向扩张系统的服务能力
- 消除单点故障
- 跨地域容灾

## 软件硬件
- LVS 
在linux内核之上的开发
- HAProxy
功能单一，负载均衡
- nginx
web服务器、缓存，很流行
- F5
支持4-7层，高度定制,F5的四层负载均衡由硬件芯片处理，不消耗CPU资源，能够处理更大的访问量。在四层负载均衡模式下，真实服务器的默认网关必须指向F5的自身内网IP。硬件都是定制的，速度肯定快，但性价比不一定好

其他软件
### Google Maglev
A Load Balancer on Commodity Servers，8-core CPU capped at 12M pps (packets per second). Maglev is not using the Linux kernel network stack which would slow it down to less than 4M pps
特性
- 绕过linux内核，直接和驱动打交道，否则会有性能损耗
- 一致性哈希实现TCP状态服务做成了无状态的
- 与网卡共享缓存，尽量减少数据拷贝
- cpu绑定，减少cache miss
- 一致性hash算法Maglev Hashing直接转发到目标机器，否则需要维护全局的五元组缓存（美团做法）,负载均衡器到后端服务器之间用的是五元组（保证连接）+第一次转发（保证会话）
- 与lvs一样，三角传
- 高性能,10Gbps line rate on each machine

### 美团MGW
仿照Maglev设计
- 使用DPDK 
- 网卡驱动之上，内核之下的一层封装

### 淘宝Tengin
看名字，改自nginx

### 京东
使用nginx模块化机制二次开发,使用nginx缓存热数据

### 其他
DISQUS, GitHub, Imgur, Instagram,使用HA

## 分类
### 交换机到负载均衡器用ECMP

### 第四层传输层
- 直接是建立连接搭桥，无视内容，效率高
- 开源IVS技术

### 第七层应用层
Apache, Lighttpd and nginx web servers 
效率相对较低，需要对协议进行处理

### DNS Load Balancing
为了保证跨IDC的负载均衡，如果是一个IDC内集群，或系统内部的负载均衡，没有必要

# HAProxy vs nginx
两个开源软件其实很像，c开发，都是单进程设计，为了性能，大量采用异步方式，多核心使用的时候分master与worker进程。

### 性能nginx不如HA（实测，差不多）
### ngix官方支持每秒5万并发，实际一般到每秒2万并发左右
### HA官方10w并发
### nginx对windows支持但不好
- 不支持完成端口
- 虽然可以启动若干工作进程运行，实际上只有一个进程在处理请求所有请求。ps:测试的过程中确实发现，8核心的机器，好几个ngnix进程，但是只有一个是工作的，
站了12%（单核爆了）的cpu，太挫了
- 一个工作进程只能处理不超过1024个并发连接。
- 缓存和其他需要共享内存支持的模块在Windows Vista及后续版本的操作系统中无法工作，因为在这些操作系统中，地址空间的布局是随机的。
### 功能nginx比HA多,开源社区维护nginx比HA多
### 负载均衡策略，nginx部分收费

# 使用与调优
## nginx

### 调优
#### worker_processes
配置使用的cpu数目和cpu绑定，无论作为代理还是web服务器，都需要根据机器的cpu信息进行配置。

#### worker_connections
events模块，配置表示每个工作进程的并发连接数，默认设置为1024，在events模块下。

#### keepalive
http_upstream_module 模块，Activates the cache for connections to upstream servers.

#### access_log
记录请求的log，默认是打开的，可以在相应模块中给指定关闭，例如http模块中关闭，但实测对性能影响并不大，关闭前最后经过压力测试验证。

## HAproxy

### 事件驱动、单一进程模型
多进程或多线程模型受内存限制 、系统调度器限制以及无处不在的锁限制，很少能处理数千并发连接

### 负载均衡算法
- roundrobin，表示简单的轮询
- static-rr，表示根据权重
- leastconn，表示最少连接者先处理
- source，表示根据请求源IP
- uri，表示根据请求的URI
- url_param，表示根据请求的URl参数
- hdr(name)，表示根据HTTP请求头来锁定每一次HTTP请求；
- rdp-cookie(name)，表示根据据cookie(name)来锁定并哈希每一次TCP请求

## LVS

### 三种工作方式
#### NET 效率一般，从实现方式上看应该与nginx和HA的4层负载效率差不多(待实测)

<img src="http://jbcdn2.b0.upaiyun.com/2017/02/cfdcc72a5d25c7c782be8675483ddbe3.png" />

#### DR 效率最高，企业中应用最多

<img src="http://jbcdn2.b0.upaiyun.com/2017/02/0a8f9ce20c8934f9e3bbcdd15d4bff7d.png" />

#### Tun 限制较多，使用不多

<img src="http://jbcdn2.b0.upaiyun.com/2017/02/3133868705ba82c6d92ebda872589cf1.png" />

# 参考
[LB 负载均衡的层次结构](http://blog.csdn.net/mindfloating/article/details/51020767)  
[HAProxy vs nginx: Why you should NEVER use nginx for load balancing!](https://thehftguy.com/2016/10/03/haproxy-vs-nginx-why-you-should-never-use-nginx-for-load-balancing/)  
[大型网站架构系列：负载均衡详解（下）](http://blog.jobbole.com/97960/)  
[nginx文档](http://nginx.org/en/docs/)  
[压力测试](http://xiaorui.cc/2016/06/26/%E8%AE%B0%E4%B8%80%E6%AC%A1%E5%8E%8B%E6%B5%8B%E5%BC%95%E8%B5%B7%E7%9A%84nginx%E8%B4%9F%E8%BD%BD%E5%9D%87%E8%A1%A1%E6%80%A7%E8%83%BD%E8%B0%83%E4%BC%98/)  
[MGW——美团点评高性能四层负载均衡](http://tech.meituan.com/MGW.html)  
[Maglev介绍](https://zhuanlan.zhihu.com/p/23826170)  
[Maglev论文](https://static.googleusercontent.com/media/research.google.com/zh-CN/pubs/archive/44824.pdf)  
[Tngine](http://tengine.taobao.org/)  
[使用 LVS 实现负载均衡原理及安装配置详解](http://blog.jobbole.com/110200/)  
