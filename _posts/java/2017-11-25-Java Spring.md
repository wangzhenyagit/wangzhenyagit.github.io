---
layout: post
title: Java Spring
category: Java相关
tags: Java
---

## OverView ##

一般说的Spring指的的Spring Framework，也是SSM中的一个S，另外一个S是Spring MVC，除了这两个项目，在Spring下还有很多的项目，如：

- Spring IO platform，在main project的第一个，解决多个包之间版本依赖问题，网状的依赖升级版本时候也可以放心了
- Spring Session，能够实现session的共享
- Spring Boot，比较火，服务器环境开始配置
- Spring Batch 批处理用的
- Spring Integration，实现了 Enterprise Integration Patterns
- Spring Shell，简单的交互程序，非常实用 http://www.baeldung.com/spring-shell-cli
- Spring for Apache Kafka，看来kafka是火的不行了，让火的不行的spring也有project支持
- Spring HATEOAS，火的不行的rest
- Spring Cloud，微服务中可用，spring博大精深额

上面的项目，Spring Boot应用比较广，Spring shell非常的实用，用于调试命令，用注解开发非常方便。

## IOC ##

IOC与AOP这两个概念不是Spring特有的，只是Spring实现了这两个概念。
> In software engineering, inversion of control (IoC) is a design principle in which custom-written portions of a computer program receive the flow of control from a generic framework. A software architecture with this design inverts control as compared to traditional procedural programming: in traditional programming, the custom code that expresses the purpose of the program calls into reusable libraries to take care of generic tasks, but with inversion of control, it is the framework that calls into the custom, or task-specific, code.

上述是wiki中对控制反转的解释，正常的流程是code来控制程序的流程，而在Ioc中，是框架来控制整个流程，当然怎么控制，是需要提前配置的。实现上也没啥难的，就是一个依赖注入，定义好各个模块的接口，然后注入能够执行的对象，让框架来跑就好了。

## AOP ##

> In computing, aspect-oriented programming (AOP) is a programming paradigm that aims to increase modularity by allowing the separation of cross-cutting concerns. It does so by adding additional behavior to existing code (an advice) without modifying the code itself, instead separately specifying which code is modified via a "pointcut" specification, such as "log all function calls when the function's name begins with 'set'". This allows behaviors that are not central to the business logic (such as logging) to be added to a program without cluttering the code, core to the functionality. AOP forms a basis for aspect-oriented software development.

上面的IOC是个principle，准则，而这AOP上升到了paradigm，范式，与面向对象是一个层面的，而IOC只是一个类型与“高内聚低耦合”的一个准则。不像面向对象这思想样博大精深，AOP相对好理解，对原来的代码不改动，但是可以增加一些功能，如日志。AOP更多的是多OOP的一个补充，不是替代。

实现AOP有几种方式，常见的一种实用AspectJ框架，是在编译的时候生成代理，而Spring AOP是在运行的时候动态生成代理类。

## spring bean 生命周期 ##

参考官方文档：[Bean scopes](https://docs.spring.io/spring/docs/3.0.0.M3/reference/html/ch04s04.html)

