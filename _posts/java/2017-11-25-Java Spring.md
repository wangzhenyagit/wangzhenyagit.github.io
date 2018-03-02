---
layout: post
title: Spring基础概念
category: Java相关
tags: Java
---

## OverView ##

文章多数参考wiki和《Spring In Action》第四版，如不说明，英文引用wiki，中文的引用《Spring In Action》。

一般说的Spring指的的Spring Framework，也是SSM中的一个S，另外一个S是Spring MVC，除了这两个项目，从[官网](https://spring.io/projects)的结构上看，Spring其实包括了很多项目：

- Spring IO platform，在main project的第一个，解决多个包之间版本依赖问题，网状的依赖升级版本时候也可以放心了
- Spring Session，能够实现session的共享
- Spring Boot，比较火，服务器环境开始配置
- Spring Batch 批处理用的
- Spring Integration，实现了 Enterprise Integration Patterns
- Spring Shell，简单的交互程序，非常实用 http://www.baeldung.com/spring-shell-cli
- Spring for Apache Kafka，看来kafka是火的不行了，让火的不行的spring也有project支持
- Spring HATEOAS，火的不行的rest
- Spring Cloud，微服务中可用 

上面的很多项目都非常实用，例如在安防行业，如果系统业务类型多，系统架构可以基于微服务，那么Spring Cloud可以实现微服务中很多特性，服务治理、API网关等，而一般Spring Cloud是基于Spring Boot进行开发，同时，安防中很多对接的工作，那么Spring Integration就能派上用场了，其他的，如果对接的系统数据量较大，但没有上一定规模，可以使用轻量级的Spring Batch代替Spark和其他Hadoop 的MR，其他的对消息队列Kafka、共享Session问题，Spring也有对应的项目实现，一般系统都会用到。

这里只讨论基本的Spring的概念，Spring虽然功能强大，但是上面所有的项目都是基于几个少许的基本概念，而且目的非常明确，引用《Spring In Action》中的一句话：

> 所有的理念都可以追溯到Spring的最根本使命上：简化Java开发。

从语言层面上看，Java相对于C/C++，开发效率已经有很大的提升，如果说Java是简化了以前类型安全的语言的开发，Spring框架在此之上更近一步简化了Java的开发，当然还有更厉害的Spring Boot —— 简化了Spring的开发。

Spring是如何简化Java开发的？Spring采用了四种策略。

> - 基于POJO的轻量级和最小侵入性编程；
> - 通过依赖注入和面向接口实现松耦合；
> - 基于切面和惯例进行声明式编程；
> - 通过切面和模板减少样板试代码；

POJO可以简单理解为没有受到约束的的一般的Java对象：
> Ideally speaking, a POJO is a Java object not bound by any restriction other than those forced by the Java Language Specification

还是很抽象，那么可以从POJO不是什么说起，POJO对象不会继承任何的接口和类，这些继承就是一些特殊的约束和限制。而且与后面说的“最小入侵性”相对应，继承是一个强的耦合，或者说是对代码的强入侵，即使不是框架，业务代码的实现上，也尽量少用继承，这个问题就不在讨论，Effective C++/Java书中提到很多。

第一种策略，个人感觉是Spring的设计出发点，可以说是简化Java开发的核心理念，后面的DI和AOP是关键策略，而第四点的使用模板甚至都不是什么核心策略，只是代码的封装而已。

基于POJO和最小侵入性编程有什么优点呢？所有的框架都是为了少写代码快速开发，那么Spirng开发的的项目，个人觉得就是，代码清爽，能够专业于核心的业务，代码量减少很多，而不是从一堆代码中，找出那么一两句核心的代码。

缺点么，框架一是会带来效率的降低，待考证验证。二是会有很多注解，但相比实现接口的框架方式比，注解明显要清新很多。

那Spring Boot又是怎么进一步简化Spring开发的呢？同样有四个特性：
> - Spring Boot Starter将很多的依赖分组进行了整合，减少了POM的配置。
> - 自动配置：合理地推测应用所需的bean并自动化配置它们；
> - 命令行接口CLI:发挥了Groovy编程语言的优势，并结合自动配置进一步简化Spring应用开发。
> - Actuator：它为Spring Boot应用添加了一定的管理特性。

第一个特性就是POM的配置上，可以简化很多POM文件；但是有代表性的是第二点，注意，这里不是Autowired（自动装配）的特性而是类似于简化了DI的“自动配置”过程，也就是会根据上下文推测，进行自动的组装。第三个特性感觉有点Java8的lambdma表达式，功能强大，但是过于简单，看编码规范要求让不让用了。第四点是一个额外的管理特性。

关于Spring和Spring Boot如何简化开发，目前看到的比较有代表性的为Spring boot中使用kafka的代码，参考[官网上的例子](https://docs.spring.io/spring-kafka/reference/htmlsingle/#_even_quicker_with_spring_boot)
，使用20行代码+三行配置，完成了kafka的写入和读取的测试验证，这如果要用kafka的java包实现，至少上百行。


## IOC ##

IOC与AOP这两个概念不是Spring特有的，只是Spring实现了这两个概念。
> In software engineering, inversion of control (IoC) is a design principle in which custom-written portions of a computer program receive the flow of control from a generic framework. A software architecture with this design inverts control as compared to traditional procedural programming: in traditional programming, the custom code that expresses the purpose of the program calls into reusable libraries to take care of generic tasks, but with inversion of control, it is the framework that calls into the custom, or task-specific, code.

上述是wiki中对控制反转的解释，正常的流程是code来控制程序的流程，而在Ioc中，是框架来控制整个流程，当然怎么控制，是需要提前配置的。实现上也没啥难的，就是一个依赖注入，定义好各个模块的接口，然后注入能够执行的对象，让框架来跑就好了。

## AOP ##

> In computing, aspect-oriented programming (AOP) is a programming paradigm that aims to increase modularity by allowing the separation of cross-cutting concerns. It does so by adding additional behavior to existing code (an advice) without modifying the code itself, instead separately specifying which code is modified via a "pointcut" specification, such as "log all function calls when the function's name begins with 'set'". This allows behaviors that are not central to the business logic (such as logging) to be added to a program without cluttering the code, core to the functionality. AOP forms a basis for aspect-oriented software development.

上面的IOC是个principle，准则，而这AOP上升到了paradigm，范式，与面向对象是一个层面的，而IOC只是一个类型与“高内聚低耦合”的一个准则。不像面向对象这思想样博大精深，AOP相对好理解，对原来的代码不改动，但是可以增加一些功能，如日志。AOP更多的是多OOP的一个补充，不是替代。

实现AOP有几种方式，常见的一种实用AspectJ框架，是在编译的时候生成代理，而Spring AOP是在运行的时候动态生成代理类。

## spring bean 默认为什是单例 ##

与此对应的另外一个问题，spring boot中的“自动配置”特性也是要求是只有一个类型的相应的对象，那如果一个类型的对象有多个该怎么办？

其实一开始也很困惑，spring boot中的自动配置，条件限制的很死啊，我如果有多个mysql的数据源，多个kafka的数据源那你这自动配置的特性就发挥不出来了啊。主要原因可能如下：

一是绝大多数的使用场景就是单例，可以确定的说，单例模式是应用最多的设计模式，甚至单例的对象，占的比例也是非常大一部分。当然，Spring可以是Prototype的。

那么如果同一个思路，spring boot的装配的对象的场景，其实很多时候，你的应用中很多的单例，而且数据源也不是有多个，kafka也不是有多个。而且目前微服务这玩意又流行，要求服务原子化，一个应用中连接多个mysql和多个kafka的情况实在是少。

所以，默认单例，自动配置，不能覆盖全部的场景，但是能覆盖大部分的使用场景。

参考官方文档：[Bean scopes](https://docs.spring.io/spring/docs/3.0.0.M3/reference/html/ch04s04.html)

