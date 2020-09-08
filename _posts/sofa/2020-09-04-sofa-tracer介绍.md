---
layout: post
title: sofa tracer介绍
category: sofa
tags: sofa
---
## 介绍
- 既然有自己的rpc，那么trace方面也会有自己对应的系统
- 这个trace相关，有个OpenTracing规范，规范的特点就是语言无关
- 而sofaTracer基于这个OpenTracing规范实现了规范中推荐的api接口、定义了相关的概念对应的类库。其中一些规范中没有明确规定的实现如traceId、spanId都有自己的一套规则
- sofaTracer只是对调用链的跟踪，并没有自己的存储、展示相关的模块，如果需要进行调用链的展示还需要使用ZipKin相关工具，sofaLookout后续会支持调用链的数据存储查询还不确定

## The OpenTracing Semantic Specification
只读了[The OpenTracing Semantic Specification](https://opentracing.io/specification/)部分，比较短简单介绍了其中几个核心的概念：data model、trace、SpanContext、span、span之间的reference、baggage、api需要实现的相关功能（没有规定实现的接口），还有个特殊的NoopTracer。

### Trace
- 这是调用链跟踪的基本概念，一个系统入口的traceId，贯穿调用链的各个节点，最后就可以根据这个节点把调用链展示出来，方便定位问题，找出系统的瓶颈
- tracerId的规则与分布式ID的规范很想，生成机器的IP、时间、递增数、进程的ID
- Trace是由多个Span构成，每个Span中都有traceId
- 规范的引用“Traces in OpenTracing are defined implicitly by their Spans. In particular, a Trace can be thought of as a directed acyclic graph (DAG) of Spans, where the edges between Spans are called References.”

### Span
- Span的google翻译“a rope with its ends fastened at different points to a spar or other object in order to provide a purchase.”，这里的purchase是hold，支撑点的意思。Span是两个节点之间的连接，比如rpc调用，是rpc client到rpc server之间的一次调用的描述
- Span中最基础的就是开始时间、结束时间、标识、和其他用于区别的tag（如userId：123456）
- Span Context， Span是有对应的一个Span Context的
- 对于SpanId，生成是特殊的一个规则，如果SpanA的id是0,那么A的子SpanB的Id是0.1、子SpanC是0.2,这样通过多级父子增加点号的方式，能够非常方便的根据ID生成一个树

### Span Context
- context与其他系统的context很类似，本地的实现应该是基于一种ThreadLocal，跨线程的时候需要使用特殊的方式来传递这个context
- 包括跨进程的唯一的id，与跨进程的一些baggage。
> - Any OpenTracing-implementation-dependent state (for example, trace and span ids) needed to refer to a distinct Span across a process boundary
> - Baggage Items, which are just key:value pairs that cross process boundaries

### baggage
- 背包，既然是背包隐含的意思就是关联没那么紧密，只是一个背着的东西，不是人的一个组成器官。在调用链中，这baggage更多的更能是携带一些与Span弱相关的，通用的、系统相关性比较大东西，在SofaTracer中把baggage分为了两类：
> SOFATracer 中将 Baggage 数据分为 sysBaggage 和 bizBaggage；sysBaggage 主要是指系统维度的透传数据，bizBaggage 主要是指业务的透传数据。

- 与Span强相关的东西可以放在Span的tag中
> - Baggage items enable powerful functionality given a full-stack OpenTracing integration (for example, arbitrary application data from a mobile app can make it, transparently, all the way into the depths of a storage system), and with it some powerful costs: use this feature with care.
> - Use this feature thoughtfully and with care. Every key and value is copied into every local and remote child of the associated Span, and that can add up to a lot of network and cpu overhead.

### Span Reference
- 这个reference可以分为两种，一种是同步的，子Span结束后或这多个子Span结束后父Span才会结束，一种是异步的，关联关系很弱，只有一个因果关系类似事件发布

### NoopTracer
> All OpenTracing language APIs must also provide some sort of NoopTracer implementation which can be used to flag-control OpenTracing or inject something harmless for tests (et cetera). In some cases (for example, Java) the NoopTracer may be in its own packaging artifact.

- 规范中要求有个默认的NoopTracer的实现，noop是no option的意思？为什么需要一个这样的默认的实现？所谓的flag-control类似rest接口中参数的required这样的flag，规范中还特意说了Java可能在打包的过程中就能完成了
- 这个noopTracer只是系统的一个可配置的入口，如果没有这种实现，系统可能就没有动态配置的能力，就是所有代码都写死的情况了
## 参考
[SOFA-tracer](https://www.sofastack.tech/projects/sofa-tracer/overview/)






