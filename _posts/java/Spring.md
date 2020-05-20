---
layout: post
title: Spring Framework
category: Java相关
tags: Spring
---

## Spring Framework总览
### 模块化
所有的模块可以直接从spring的git上看，一共20多个，https://github.com/spring-projects/spring-framework。

- 最基础的是spring-core、spring-bean、spring-context
- 有一些用的不多的spring-expression、spring-oxm
- 整合数据库的spring-jdbc、spring-orm（jpa）
- 整合了消息的spring-jms，这个jms就是java的jms规范，当然很多消息队列没有实现jms规范像kafak、amq，所以还有个spring-messaging，想统一消息的实现包括jms
- spring-tx，事物
- web相关