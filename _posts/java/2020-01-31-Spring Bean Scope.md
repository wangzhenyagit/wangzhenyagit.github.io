---
layout: post
title: Spring Bean Scope And LifeCycle
category: Java相关
tags: spring
---

## Singleton
- 是否为Singleton的信息保存在BeanDefinition和FactoryBean里面
- bean默认为Singleton方式

## prototype
- 每次依赖查找、注入都是新的实例，包括Collections里面的
- spring没有办法管理prototype的Bean的生命周期，会执行@PostConstruct但不会回调@PreDestroy，如果想实现destroy功能可以实现DisposableBean这个接口，并在使用的上线文中进行显示调用

## request、session
- 这中方式目前会逐渐边缘化，目前发展趋势为前后端分离

## 自定义Scope
- 自定义Scope可以通过实现Scope接口，并注册到容器中的方式
- 实现的时候get时会给一个BeanFactory作为入参，可以让用户自定义返回。单例也是通过BeanFactory返回前增加处理来实现？至少后面通过api注册的应该不是的。

## singleton bean是否在同一个应用中是唯一的？
- 否。一个应用可以有多个上下文
- 同理，一个static字段在应用中也不是唯一的，一个应用中可以有多个classLoader。