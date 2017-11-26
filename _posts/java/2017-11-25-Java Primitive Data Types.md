---
layout: post
title: Java Primitive Data Types
category: Java相关
tags: Java
---

“primitive data”从字面上看，是原始数据类型（wiki上的中文也是这样翻译），个人觉得这样翻译更形象，而很多场景中，多翻译成基础数据类型。下面英文的引用大多参考Oracle的官方文档:[The Java™ Tutorials](https://docs.oracle.com/javase/tutorial/java/nutsandbolts/datatypes.html)

> A primitive type is predefined by the language and is named by a reserved keyword. Primitive values do not share state with other primitive values. 

上面的表述也表明了，原始数据类型有原子性，与复合类型相对应，复合类型都是由多个原子类型组成的。Java中有八个原始数据类型，如wiki上说的，"Basic primitive types are almost always value types."，Java中的八个都是值类型，而且一半都是表示整数的值类型，byte、short、int、long，这四个从一个字节到八个字节的整数，外加上两个浮点float和double，其中float是四个字节，double是八个字节，最后就是boolean和char，特别注意的是，这个char，在C中只是一个字节，而在java中为2个字节。

> The char data type is a single 16-bit Unicode character. It has a minimum value of '\u0000' (or 0) and a maximum value of '\uffff' (or 65,535 inclusive).

int和long的对应类Integer和Long是能够保存unsigned int 和 unsigned long的，而且有对应的方法。

另外，对于Stirng这个类型，官网是这么说的，大概也就也就是说，理论上String不是基础数据类型，但是从语言支持的角度上可以把String当作一个基础数据类型。

> The String class is not technically a primitive data type, but considering the special support given to it by the language, you'll probably tend to think of it as such.

在C中，long的占用几个字节是与编译器和操作系统是32还是64位平台有关系的，而Java中，都有虚拟机了，当然会统一了，不论是32还是64位的虚拟机，int是四个字节，而long是八个字节。