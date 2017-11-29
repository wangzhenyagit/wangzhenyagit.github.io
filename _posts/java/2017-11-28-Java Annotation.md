---
layout: post
title: Java Annotations
category: Java相关
tags: Java
---

## Annotation是什么 ##
在使用Spring的时候经常会用到这个注解，注解是什么东西？参考：[The Java™ Tutorials Annotations](https://docs.oracle.com/javase/tutorial/java/annotations/index.html)，在网上看了不少资料，还是这官方英语的简单又全面，以下英文的引用不特别说明都是引用上述文档。

> Annotations, a form of metadata, provide data about a program that is not part of the program itself. Annotations have no direct effect on the operation of the code they annotate.
> 
> Annotations have a number of uses, among them:
> 
> - Information for the compiler — Annotations can be used by the compiler to detect errors or suppress warnings.
> - Compile-time and deployment-time processing — Software tools can process annotation information to generate code, XML files, and so forth.
> - Runtime processing — Some annotations are available to be examined at runtime.

注解是一个种元数据，最常见的例子就是@overwrite注解，对程序的运行没有直接的影响，但是编译器会检查，这被注解的方法的名称与参数父类是否有，如果没有，会报错的。在C++中，一个经验就是如果要重写方法，直接去父类方法声明中拷贝一份，否则错拼一个字母，就是个坑。这也是上面提到的第一点用处“Information for the compiler”。

## How To Declaring an Annotation Type ##
有个特殊的格式，@interface ，如下声明：

```
// import this to use @Documented
import java.lang.annotation.*;

@Documented
@interface ClassPreamble {
   String author();
   String date();
   int currentRevision() default 1;
   String lastModified() default "N/A";
   String lastModifiedBy() default "N/A";
   // Note use of array
   String[] reviewers();
}
```

上述的Documented叫meta-annotations，注解的元数据，这个Documented的注解就能让ClassPreamble的内容在使用javadoc的时候导出来。这也是开头提到的第二点用处“Software tools can process annotation information to generate code, XML files, and so forth.”。

## Annotation Types ##
预定义的注解可以分为两类：
>  Some annotation types are used by the Java compiler, and some apply to other annotations（meta-annotations）.

其实预定义的注解也没有几个，Java编译器默认支持的也就那几个，@Deprecated、@Override、@SuppressWarnings、@SafeVarargs、@FunctionalInterface，然后其他的Annotations都是meta-annotations，@Retention、@Documented、@Target、@Inherited、@Repeatable。

如果说注解只有上面的这几个，就个Override和Deprecated还算实用点，那这存在价值不是很大，但是接触过Spring的都知道，到处都是注解，而且用起来很方便，开头提到的第三点，“Runtime processing”，感觉优点像实现代码实现的功能，例如文中举的一个实现定时器的例子：

> For example, you are writing code to use a timer service that enables you to run a method at a given time or on a certain schedule, similar to the UNIX cron service. Now you want to set a timer to run a method, doPeriodicCleanup, on the last day of the month and on every Friday at 11:00 p.m. To set the timer to run, create an @Schedule annotation and apply it twice to the doPeriodicCleanup method. The first use specifies the last day of the month and the second specifies Friday at 11p.m., as shown in the following code example:

```
@Schedule(dayOfMonth="last")  
@Schedule(dayOfWeek="Fri", hour="23")  
public void doPeriodicCleanup() { ... }  
``` 

这样写起来不要太简单。

## Type Annotations ##
本来这也没啥好说的，但是是Java8加上的新特性，而且感觉也很实用。以前注解只是用于声明上的，但JAVA8就能用于任何的type之上，主要能通过这种方式提供更强的类型检查功能，尽早的发现问题，比如文中举的例子：

> For example, you want to ensure that a particular variable in your program is never assigned to null; you want to avoid triggering a NullPointerException. You can write a custom plug-in to check for this. You would then modify your code to annotate that particular variable, indicating that it is never assigned to null. The variable declaration might look like this:
> 
> @NonNull String str;
> 
> When you compile the code, including the NonNull module at the command line, the compiler prints a warning if it detects a potential problem, allowing you to modify the code to avoid the error. After you correct the code to remove all warnings, this particular error will not occur when the program runs.


