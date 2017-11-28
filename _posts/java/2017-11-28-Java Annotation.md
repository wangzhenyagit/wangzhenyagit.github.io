---
layout: post
title: Java Annotations
category: Java相关
tags: Java
---

## Annotation是什么 ##
在使用Spring的时候经常会用到这个注解，一直有个疑问，那些配置文件写在代码里面有什么好的呢？万一配置需要修改了岂不是还要在重新编译？这问题开个头，先看下这注解是什么东西，参考：[The Java™ Tutorials Annotations](https://docs.oracle.com/javase/tutorial/java/annotations/index.html)

> Annotations, a form of metadata, provide data about a program that is not part of the program itself. Annotations have no direct effect on the operation of the code they annotate.
> 
> Annotations have a number of uses, among them:
> 
> - Information for the compiler — Annotations can be used by the compiler to detect errors or suppress warnings.
> - Compile-time and deployment-time processing — Software tools can process annotation information to generate code, XML files, and so forth.
> - Runtime processing — Some annotations are available to be examined at runtime.

注解是一个种元数据，最常见的例子就是@overwrite注解，对程序的运行没有直接的影响，但是编译器会检查，这被注解的方法的名称与参数父类是否有，如果没有，会报错的。在C++中，一个经验就是如果要重写方法，直接去父类方法声明中拷贝一份，否则错拼一个字母，就是个坑。这也是上面提到的第一点用处“Information for the compiler”。