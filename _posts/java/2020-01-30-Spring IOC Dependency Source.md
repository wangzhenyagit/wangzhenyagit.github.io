---
layout: post
title: Spring IOC Dependency Source
category: Java相关
tags: spring
---

## 依赖查找的来源
- 主要为BeanDefination和通过单例api增加的bean

## Spring内建的bean（BeanDefination方式）
- ConfigerationClassPostProcessor对象，处理配置类
- AutowireAnnotationBeanProcessor对象，处理@Value和@Autowire
- CommonAnnotationBeanProcessor对象，处理@PostConstruct、@Resource等
- EventListenerMethodProcessor对象，处理@EventListenor
- BeanDefination主要注册的逻辑在AnnotationConfigUtils中的registerAnnotationConfigProcessors中进行注册，调用入口是在AnnotatedBeanDefinitionReader的构造函数中，这个reader是AnnotationConfigApplicationContext中的一个依赖。

## Spring内建的单例
- Environment
- LifecycleProcessor
- ApplicationEventMulticaster
- 而单例主要是在AbstractApplicationContext#prepareBeanFactory中由refresh触发在prepare中进行触发调用的。
- 注册为BeanDefination与直接注册的Sigleton两者有什么区别？主要从功能上看，一般sigleton是一个独立的功能性的与生命周期无关的特性的bean如Environment，而注入的BeanDefination一般是需要处理一个生命周期或者类中的一些特殊的注解，需要在Bean生命周期中进行一些处理。从特性上看，BeanDefination是有一些元数据配置，相对来说控制更加灵活，而单体的方式就直接注入个对象实例。

## 单例的注册
- 单例注册方式比较简单，没有BeanDefination的元数据，直接调用DefaultSingletonBeanRegistry#registerSingleton进行的注册，存储单独于BeanDefination，有自己的map存储
- 由于是外部注入的对象，所以单例对象的生命周期并不是由Spring管理
- 由于是外部直接注入的对象，所以也没有必须是POJO不要求必须有无参数的默认构造函数，get、set方法


## 依赖注入和依赖查找的来源不同
- 依赖注入的对象比依赖查找的对象会多4个类型，2个对象（4个中3个是一样的）。在AbstractApplicationContext#prepareBeanFactory中有以下逻辑，也就是说这几个类型只能通过依赖注入方式获得，而不能通过查找获得，有单独的存储的地方不在BeanFactory内
```
// BeanFactory interface not registered as resolvable type in a plain factory.
// MessageSource registered (and found for autowiring) as a bean.
beanFactory.registerResolvableDependency(BeanFactory.class, beanFactory);
beanFactory.registerResolvableDependency(ResourceLoader.class, this);
beanFactory.registerResolvableDependency(ApplicationEventPublisher.class, this);
beanFactory.registerResolvableDependency(ApplicationContext.class, this);
```
## registerResolvableDependency
> Register a special dependency type with corresponding autowired value.
> This is intended for factory/context references that are supposed to be autowirable but are not defined as beans in the factory: e.g. a dependency of type ApplicationContext resolved to the ApplicationContext instance that the bean is living in.
> 
> Note: There are no such default types registered in a plain BeanFactory, not even for the BeanFactory interface itself.

者resolvable的意思是可分辨的，意思有点类似special，而且目的说的很明确就是注册一些可以autowireable但是没有在BeanFactory中没有defined。通过getBean拿不到。为什么需要这样？通过看注册的对象可知，注册的对象是BeanFactory和ApplicationContext，难道是为了防止循环引用？BeanFactory注册自己到自己中，把BeanFactory的持有者ApplicationContext注册到BeanFactory中可能有些问题。

# 外部化配置作为依赖来源
- 非常规的的spring对象依赖来源
- 限制：无生命周期管理，无法实现延迟初始化，无法通过依赖查找

