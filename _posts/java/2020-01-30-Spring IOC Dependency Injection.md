---
layout: post
title: Spring IOC Dependency Injection
category: Java相关
tags: Java
---

## 异常BeansException
这个异常是个RuntimeException，在spring中大多数都是RuntimeException，其实这很正常，大多数的异常是不可恢复的，没有必要弄个checked异常，对于使用者来说非常不友好。无法使用于调用链的方式。

## ObjectFactory 与 BeanFactory区别

- 这两个都是spring的接口，并且都能进行依赖的查找
- ObjectFactory只关注一个或一种类型bean的查找，（可以直接配置个ObjectFactory并且关联个bean），本身并不具备依赖查找的能力，能力由BeanFactory输出。
- BeanFactroy提供了单一、集合、层次等多种类型的查找。比如DefaultListableBeanFactory这个实现，从名字listable上看就可以查找集合的bean，还有目前最常用的AnnotationConfigApplicationContext，也是BeanFactory，光实现了BeanFactory的类就有几十个。

## AnnotationConfigApplicationContext
- 虽然实现BeanFactory的接口非常多，但大家在调用getBean的时候，都是通过类似代理的方式，最后通过一个DefaultListableBeanFactory具体的BeanFactory来进行getBean的，bean的元数据信息和bean的实例信息都是比较多，的没有必要每个BeanFactory都来copy一部分，copy后还有会数据一致性问题。
- BeanFactory.getBean是线程安全的，很多操作多有synchronized保证安全

## setter、construct依赖注入
- 手动注入的方式有xml、annotation、api（BeanDefination通过addPropertyReference方式）
- setter自动注入方式有byName和byType，注意spring并不推荐此方式，autowire默认是no

## 字段注入@Autowired
- @Autowired的方式能直接注入特定的字段
- 只注入实例的字段，忽略static字段
- 
## 方法注入
- @Autowired直接作用于方法上即可，与方法名字无关。
- 方法注入感觉可以在方法中加入自己的一些定制的配置，更加灵活
- 从方法注入到直接字段注入，相比字段注入写起来更加简单，有些像lambda表达式的简化，从一个方法体简化到() -> {}这种方式，为了更简单的代码开发
- @Bean也是一种方法注入，只不过返回的实例对象是给了容器，而不是当前的实例字段

## 接口回调注入（Aware接口）
- aware接口是个系列，最常见的就是BeanFactoryAware和ApplicationContextAware
- 对于使用者来说，虽然是个接口，只是对与框架自己是扩展开放的，对于使用者来说并不是开发的，只能等spring自己扩展aware的系列

## 依赖注入的选择
- spirng的作者是推荐构造器注入的。如果构造器参数很多的场景下，使用构造器注入就比较麻烦（当然可能也是类设计的问题），但是构造器的参数顺序是确定的，如果在内部的多个依赖有顺序相关性，setter注入是表达不了这种顺序的，是有可能出问题的。从 Spring 4.3 开始, 在只有一个构造器的类上, @Autowired注解可以不需要显示指定. 这是特别优雅的, 因为它使得类将不携带任何容器注释.这也能看出，spring对构造器注入的倾向。
- @Autowired在字段上这种方式已经不在推荐，并且在spring中在慢慢淘汰。个人感觉最主要的是这种方式对于类的使用来说是不够清晰的，看到这个类，或者初始化这个类的时候，都不知道需要依赖的其他的实例是什么，还需要进入类中去找@Autowired注解，写起来方便，读起来费劲。
- 方法注入一般是@Bean的方式用的比较多，自定义组装的过程。

## Qualifier
- 者个单词意思是“有资格的”、“被官方认证的”，在注入的时候可以增加名字的限定
- 这个注解也能放在@Bean后，用在Bean的声明上，对对象进行分组
- 这个注解还能作用在注解上，@Target中有ElementType.ANNOTATION_TYPE，这中方式相当于注解的派生，作用域注解上时，被标记的注解相当于一个新的分组。在spring cloud中的@LoadBalance注解就是这种方式

## ObjectProvider与ObjectFactory
- ObjectProvider能提供相对安全的、单一的或集合的延迟的注入，需要通过getObject方式二次获取，用于对一些非必要的bean的注入（比如正常流程中大概率用不到的bean），支持stream方式，更加现代化，在spirng boot中有大量的使用这个类
- ObjectFactory也能延迟方式注入，但是如果想注入集合只能把泛型定义为集合类型

## 依赖注入处理过程
- 入口-DefaultListableBeanFactory#resolveDependency 解析依赖的入口
- 依赖描述符-DependencyDescriptor，也是个元信息的对象，与BeanDefination类似，只是这个类是用来表示依赖的元信息，虽然看起来没什么元信息比如下代码：@Autowaired User user;但是其实有不少默认信息的:必须（required=true）、实时注入（eager=true)、通过类型（User.class）、字段名称（"user"）、是否首要（primary = true)，还有比如@Lazy这些注解的信息，也是在这descriptor中的。
- 绑定候选对象处理器-AutowireCandidateResolver，这个是策略接口，因为一个类型的对象可能在上线文中有多个，所以需要这个resolver，Strategy interface for determining whether a specific bean definition qualifies as an autowire candidate for a specific dependency.

## @Autowired过程
- 元信息解析，通过在AbstractAutowireCapableBeanFactory（具有自动注入能力的beanFacory）进行createBean的时候，通过框架调用InstantiationAwareBeanPostProcessor（AutowiredAnnotationBeanPostProcessor）的postProcessProperties方法触发的。这properties名字应该是从xml中配置而来的，为什么不叫field呢，应为xml中的对象都是string的，没有经过类型转换，更符合property的定义。postProcessProperties这个方法调用findAutowiringMetadata，找到要被注入对象的InjectionMetadata，这个元信息是个单纯的被注入对象的元信息，与DependencyDescriptor不同，没有被注入对象的一些描述信息，是通过DependencyDescriptor找到对象后，对象的元信息。
- 依赖查找，在inject过程中触发
- 依赖注入，在找到InjectionMetadata后会调用它的inject方法，在inject的过程中触发依赖查找的过程

## @Inject
- 这个注解是jsr330中的特性，spring处理的过程和autowire基本类似
- 这注解支持static注入，Identifies injectable constructors, methods, and fields. May apply to static as well as instance members.

## CommonAnnotationBeanPostProcessor
- 处理java的通用的注解如@Resource，并且处理过程中还处理了生命周期中javax.annotation.PostConstruct和PreDestroy注解
- 与AntowiredAnnotationBeanPostProcessor相比，CommonAnnotationBeanPostProcessor多实现了个InitDestroyAnnotationBeanPostProcessor，并在构造的时候会设置initAnnotationType（默认为PostConstruct和PreDestroy），设置了这个后InitDestroyAnnotationBeanPostProcessor就会在生命周期中进行处理,代码如下，所以这@PostConstruct标注的方法是先与Initialization先执行的
```
	public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
		LifecycleMetadata metadata = findLifecycleMetadata(bean.getClass());
		try {
			metadata.invokeInitMethods(bean, beanName);
		}
```

## 注入和查找的来源是否相同？
- 依赖查找的来源仅仅限于BeanDefinition及单例对象
- 而依赖注入的来源还有Resolvable Dependency及@Value的外部化配置

## 在容器初始化完成后不在允许BeanDefinition注册？
- 可能原因是因为需要的这类场景比较少，实现的化比较复杂。BeanDefinition方式提供了很多特性都是在注册过程中进行控制的，你这过程都结束了，也没啥能控制的了就不要通过这方式进行注册了。而且spring提供了通过外部单例方式注册的方式，简单没有元信息，能满足大多数场景需求。