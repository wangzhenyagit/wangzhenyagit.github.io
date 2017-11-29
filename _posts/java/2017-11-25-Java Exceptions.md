---
layout: post
title: Java Exceptions
category: Java相关
tags: Java
---

## 异常是什么 ##

[What Is an Exception?](https://docs.oracle.com/javase/tutorial/essential/exceptions/definition.html)

## 为什么要有异常机制？ ##
C语言是没有异常机制的，从这点上就能看出来这异常机制的效率是不高的，那异常机制存在的意义是什么？

很多与业务层较远的通用组件遇到的问题，通用组件也不知道如何处理，只能通知下上层使用者出现的问题。在C中可以通过返回错误码方式解决，具体信息可以通过错误码查询的方式查看。

异常机制能够使开发更加专注于正常的逻辑流程，处理非正常逻辑的地方可以从核心代码中搬出来，这样开发速度快，代码理解容易，对程序员友好。比如一个很复杂的流程，调用了很多方法，可以在所有调用方法的最外面进行异常处理，这样核心流程相对比较清晰。但什么是核心流程？可能有的异常处理也是属于核心流程的，这个时候，就不要攒着了，该处理就先处理了。这是异常机制的一个设计初衷。

另外，都向外抛异常对象了，那还可以对对象进行抽象、分类，也符合人类的理解方式，给问题先打标签，分成有限的几类。还能够根据类型处理的流程不一样，如check exception和uncheck exception。

异常处理的一个最佳实践，异常信息就不要给用户看了，像一般的web网站，如果有异常问题，直接跳转到要给友好的错误页面就可以了。

在Oracle网站上有说异常机制的优点的文章:[Advantages of Exceptions](https://docs.oracle.com/javase/tutorial/essential/exceptions/advantages.html)，几点如下：

> - Advantage 1: Separating Error-Handling Code from "Regular" Code
> - Advantage 2: Propagating Errors Up the Call Stack
> - Advantage 3: Grouping and Differentiating Error Types

其中第一点和第三点上面已经说到了，第二点意思是说在捕获到异常的时候，会把调用的栈信息提供出来，这样定位问题就非常的方便。

## Unchecked Exceptions ##
这Unchecked Exception也叫Runtime Exception，默认的继承自Exception接口的都是checked exception，而继承自Runtime Exception的维Unchecked Exception。为啥叫checked？谁check？对于checked的异常，编译器编译的时候会检查的，你的方法外面有没有处理了这异常。

通常来说，Checked Exception如果客户端捕获到是知道如何处理的，而Runtime Exception是客户端处理不了的，所以其实捕获到也没有什么意义，如果自定义异常，那么绝大多数应该是checked exception类型的。只有一种情况比较特殊，用户使用方法的方式不对，比如传入的参数为空指针，可以直接抛个unchecked的NullPointerException异常。

对于checked exception是需要在声明方法的时候一起说明的，而且是客户端必须捕获的，下面说明了原因，这些异常被认为与参数与返回值一样，是方法的一个部分，如果要使用这方法，必须处理这些异常。

> Why did the designers decide to force a method to specify all uncaught checked exceptions that can be thrown within its scope? 
> 
> Any Exception that can be thrown by a method is part of the method's public programming interface. Those who call a method must know about the exceptions that a method can throw so that they can decide what to do about them. These exceptions are as much a part of that method's programming interface as its parameters and return value.

java中的几个unchecked exception的类型：

- NullPointerException
- ArrayIndexOutOfBound
- IllegalArgumentException
- IllegalStateException
- NumberFormatException
- ArithmaticException

checked exception 类型：

- IOException
- SQLException
- DataAccessException
- ClassNotFoundException
- InstantiationException

## 参考 ##
[Java 提高篇(四) --- 异常处理](http://liuzxc.github.io/blog/java-advance-04/)  
[Unchecked Exceptions — The Controversy](https://docs.oracle.com/javase/tutorial/essential/exceptions/runtime.html)

