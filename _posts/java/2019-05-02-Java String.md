---
layout: post
title: Java String
category: Java相关
tags: Java
---

## String 

String是平时使用最多的一个类，虽然不是基础的数据类型，但是使用上和基础数据类型是一样的。作为函数的参数的时候，感觉上是在传值，而不是传引用，比较的时候也可以像基础数据类型一样，比较的是值，而不是内存地址，为什么呢？

### String是constant的

> Strings are constant; their values cannot be changed after they
> are created. String buffers support mutable strings.
> Because String objects are immutable they can be shared.

存储数据的char数据是final的：private final char value[];

这个特性决定了，是不能修改的，在拼接的时候会生成新的类，值相同的对象会重复利用，为什么这样设计呢？

> 把常见应用进行堆转储（Dump Heap），然后分析对象组成，会发现平均 25% 的对象是字符串，并且其中约半数是重复的。如果能避免创建重复字符串，可以有效降低内存消耗和对象创建开销。

就是因为String用的频繁，且重复性很高，如果共用相同值的对象，会减少内存的占用。

还有，首先String是个对象，如果一个对象是常量，那么就不会有多线程安全的问题，不用每次处理的时候进行锁操作，对应的比StringBuffer，所有的操作上都有个synchronized，那么从一般的经验来看，效率上肯定不如String，而且大多数的场景下，期望的场景是传值，如果没有String那么可能代码中会出现很多new StringBuilder(sb)的代码。所以，String不光没有线程安全的问题，也方便了值传递的方式。

### String的优化

> 在运行时，字符串的一些基础操作会直接利用 JVM 内部的 Intrinsic 机制，往往运行的就是特殊优化的本地代码，而根本就不是 Java 代码生成的字节码。Intrinsic 可以简单理解为，是一种利用 native 方式 hard-coded 的逻辑，算是一种特别的内联，很多优化还是需要直接使用特定的 CPU 指令

JVM对于String也有一些优化，优化的方式也没啥新奇，跳过层层的封装，直接使用高效的可能丑陋的代码，调用特定的指令。

此外，String的constant，对于编译器来说可以进行很多优化，比如：
```
String s1 = "abc";
String s2 = "a"+"b"+"c";
System.out.println(s1 == s2); // true
```
在编译阶段可以毫不犹豫的把s2直接替换成abc。

### String的演进

> 如果你仔细观察过 Java 的字符串，在历史版本中，它是使用 char 数组来存数据的，这样非常直接。但是 Java 中的 char 是两个 bytes 大小，拉丁语系语言的字符，根本就不需要太宽的 char，这样无区别的实现就造成了一定的浪费。密度是编程语言平台永恒的话题，因为归根结底绝大部分任务是要来操作数据的。

> 其实在 Java 6 的时候，Oracle JDK 就提供了压缩字符串的特性，但是这个特性的实现并不是开源的，而且在实践中也暴露出了一些问题，所以在最新的 JDK 版本中已经将它移除了。

> 在 Java 9 中，我们引入了 Compact Strings 的设计，对字符串进行了大刀阔斧的改进。将数据存储方式从 char 数组，改变为一个 byte 数组加上一个标识编码的所谓 coder，并且将相关字符串操作类都进行了修改。另外，所有相关的 Intrinsic 之类也都进行了重写，以保证没有任何性能损失

确实是问题，就像Unicode铁打的4个字节，而UTF-8的变长编码，通过减少编码长度，降低的不仅仅是空间，上文所说，String会占用一大部分内存，而内存的拷贝复制都是有时间成本的，优化后的String不仅仅是空间占用少，而且是更快的速度。

## 参考
[String、StringBuffer、StringBuilder有什么区别？](https://time.geekbang.org/column/article/7349)  
