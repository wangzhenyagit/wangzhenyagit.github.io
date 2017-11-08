---
layout: post
title: 消息队列（二）AMQ
category: 消息传递
tags: 消息队列
---

AMQ是个轻量级的消息队列，下载下来几十兆，马上就可以用，还自带web管理端，而且功能比较多。一般的小项目需求都能满足。在消息队列（一）中说明了消息队列设计的几个问题，下面针对AMQ的处理方式关键点进行下分析。

# 设计问题
## [Slow Consumers](http://activemq.apache.org/slow-consumers.html)
慢消费问题会导致topics会有大量缓存，对于需要持久化的topic，可以直接进行持久化操作。对于Non-Durable Topics处理方式有如下：
- block/slow the producer
- drop the slow consumer
- spool messages to disk
- discard messages for the slow consumer

AMQ中的默认配置是>block producers until the slow consumer catches up (for non-durable topics here).

对于此问题，可以专门设立慢消费的侦测程序Slow Consumer Detector

## [Messages redeliver](http://activemq.apache.org/message-redelivery-and-dlq-handling.html)
消息重发出现的场景如下
- A transacted session is used and rollback() is called.
- A transacted session is closed before commit is called.
- A session is using CLIENT_ACKNOWLEDGE and Session.recover() is called.
- A client connection times out

此外对于一直从发失败的消息，可以看作Poison ack（message was considered a poison pill），处理方式如下：
>Broker then takes the message and sends it to a Dead Letter Queue so that it can be analyzed later on.
>By default, ActiveMQ will not place undeliverable non-persistent messages on the dead-letter queue. 
>By default, ActiveMQ will never expire messages sent to the DLQ but from v5.12, the deadLetterStrategy supports an expiration attribute where the value is in milliseconds.

## [Supporting IO Streams](http://activemq.apache.org/supporting-io-streams.html)
support for streaming files over ActiveMQ of any arbitrary size

## [Message Store](http://activemq.apache.org/amq-message-store.html)
可通过日志方式记录消息记录，与log4xx雷系，可以配置单文件大小（typically 32mb in size），压缩，定时清理，还支持异步IO。

## [Dispatch Policies](http://activemq.apache.org/dispatch-policies.html)
prefetch value预先读取的方式，[SEDA](http://www.eecs.harvard.edu/~mdw/proj/seda/),as much work as possible asynchronously.
Async Sends，The cases that we are forced to send in sync mode are when persistent messages are being sent outside of a transaction.

# 基础配置
## AcknowledgeMode
- AUTO_ACKNOWLEDGE，回调给上层后自动ack，上层需要保证消息处理的。
- CLIENT_ACKNOWLEDGE，需要客户端主动调用ack。
- SESSION_TRANSACTED，在一个session上，也可以调用transcate方法提交。
- INDIVIDUAL_ACKNOWLEDGE
- DUPS_OK_ACKNOWLEDGE

## 消息优先级
0到9的优先级路线级别，0是最低

## 消息正文的类型
- StreamMessage -- Java原始值的数据流 
- MapMessage--一套名称-值对
- TextMessage--一个字符串对象
- ObjectMessage--一个序列化的 Java对象
- BytesMessage--一个未解释字节的数据流

# 高级功能
## Exclusive Consumer
consumer可以配置优先级。可以实现类似负载均衡，主备的功能。

## JMS Selectors
实现高级订阅过滤

## Composite Destinations
复制转发消息

## Pending Message Limit Strategy
预读策略，提速

## [Mirrored Queues](http://activemq.apache.org/mirrored-queues.html)
指定一个queue为外一个queue的镜像，这样能够监控queue中的消息，而不必在queue端在打日志了。

## Wildcards
支持正则表达式方式订阅

## Strict Order Dispatch Policy
对于特殊的场景，多个Producer保证多个consumer接收到是顺序一致的，AMQ中默认是不开启的

# MYQA
Q：transacted session 调用 commit() 与CLIENT_ACKNOWLEDGE Session 调用 acknowledge()对消息ack 区别？

A: 作用对象不一样，acknowledge是确认单条消息，commit可以是对一个session的一批消息，session也有acknowledge方法是对session没有ack的消息全部ack.

Q: 如果一个queue有多个订阅者，消息需要ack，默认配置是其中一个确认后即可？

A:> A JMS Queue implements load balancer semantics. A single message will be received by exactly one consumer. If there are no consumers available at the time the message is sent it will be kept until a consumer is available that can process the message. 

多个订阅者情况下是负载均衡机制

Q: producer与consumer都建立session都可以设置类型，不一致会怎样？

A: producer的session与consumer的session没有关系

# 参考
[AMQ官网QA](http://activemq.apache.org/faq.html)
