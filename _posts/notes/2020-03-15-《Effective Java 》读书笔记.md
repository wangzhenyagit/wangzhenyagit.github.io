---
layout: post
title: 《Effective Java》读书笔记
category: 读书笔记
tags: 笔记
---
 Effective C++ 好久之前的度过，这Effective Java中文的读起来好费劲，参考英文的，做个笔记。 已经有第三版了，看的二版的中文，三版的英文

## Chapter 02 - Creating and Destroying Objects
### Item 1 - Consider static factory methods instead of constructors
先说结论：
> In summary, static factory methods and public constructors both have their uses, and it pays to understand their relative merits. Often static factories are preferable, so avoid the reflex to provide public constructors without first considering static factories.
 
创建对象，不要第一时间想到使用构造函数，而静态工厂方法是第一选择，理由如下：

> - One advantage of static factory methods is that, unlike constructors, they have names.
  
这么有名的一本书，第一条建议的第一个理由是“have names”，深以为然。书中的例子，BigInterger.probablePrime这个静态方法，一看就是返回个素数。特别是当参数的签名一样的时候，但是要想创建返回不同类型的构造器，那只能通过调整参数的顺序，然后文档注释的方式。

> - A second advantage of static factory methods is that, unlike constructors, they are not required to create a new object each time they’re invoked.   

比如Boolean对象的valueOf方法，不会每次都返回创建一个新的对象。这还带来了另外一个特性，如枚举的时候，使用==也能够进行比较。

> - A third advantage of static factory methods is that, unlike constructors, they can return an object of any subtype of their return type. 

最常见的场景是java的Collections的工具类，能方便创建很多集合。

> - A fourth advantage of static factories is that the class of the returned object can vary from call to call as a function of the input parameters.

这个特性也是由第三个的优点的扩展，调用者不用关心到底返回的是什么对象，甚至是透明的，在后续接口升级的时候替换个对象对调用者也是无感知的。

>- A fifth advantage of static factories is that the class of the returned object need not exist when the class containing the method is written

有些不是很明白。。

缺点一是严格使用工厂方法创建，而构造方法是私有的，就没有方法继承；二是，不好找！
>- The main limitation of providing only static factory methods is that classes without public or protected constructors cannot be subclassed. 
>- A second shortcoming of static factory methods is that they are hard for programmers to find. 

常用的静态方法的名字：  

> - from—A type-conversion method that takes a single parameter and returns a corresponding instance of this type, for example:
> 
Date d = Date.from(instant); 

类型转换用的，一般是一个参数。

> - of—An aggregation method that takes multiple parameters and returns an instance of this type that incorporates them, for example:
> 
Set<RankfaceCards = EnumSet.of(JACK, QUEEN, KING);

多个参数，返回集合的场景。

> - instance or getInstance—Returns an instance that is described by its parameters (if any) but cannot be said to have the same value, for example:
> 
StackWalker luke = StackWalker.getInstance(options); 

> - create or newInstance—Like instance or getInstance, except that the method guarantees that each call returns a new instance, for example:
> 
Object newArray = Array.newInstance(classObject, arrayLen); 

与instance和getInstance不同，这个create与newInstance都是返回一个new instance。
> 
- getType—Like getInstance, but used if the factory method is in a different class. Type is the type of object returned by the factory method, for example:
> 
FileStore fs = Files.getFileStore(path); 

>- newType—Like newInstance, but used if the factory method is in a different class. Type is the type of object returned by the factory method, for example:
> 
BufferedReader br = Files.newBufferedReader(path); 


> - type—A concise alternative to getType and newType, for example:

List<Complaintlitany = Collections.list(legacyLitany);

后面者三个返回的类型与调用的不一样。

### Item 2: Consider a builder when faced with many constructor parameters

结论：
> The Builder pattern is a good choice when designing classes
whose constructors or static factories would have more than a handful of
parameters, especially if many of the parameters are optional or of identical type.

#### 使用构造器或者工厂的问题：

> -  the telescoping constructor pattern works, but it is hard to write
client code when there are many parameters, and harder still to read it.

场景就是构造方法或者静态工厂方法的时候，如果参数太多，并且一些可选一些必选的场景。

#### 缺点
- 需要额外先构造出一个builder，对于性能要求高的场景可能需要注意
- 使用起来调用链比较长

### Item 3: Enforce the singleton property with a private constructor or an enum type

## 参考：
[Effective Java - 3rd Edition Notes
](https://github.com/ekis/effective-java-3rd-edition)  
[3版电子书](https://pan.baidu.com/s/1mJx5ZrOD_RPjf3ghQnBV5g)