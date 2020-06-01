---
layout: post
title: Spring Bean LifeCycle
category: Java相关
tags: spring
---

## BeanDefinition元信息配置解析
可以通过多种BeanDefinitionReader读取BeanDefinition的配置（注解、xml、properties），并解析生成BeanDefinition，然后进行registry。

这个过程更像一个“解析”的过程，所有的信息都有（在注解、xml、properties），而不是个create的过程，所以名字叫BeanDefinitionReader，配置了几个就会读出来几个，并没有实例数目的增加。

Reader需要配置个BeanDefinitionRegistry，而默认的DefaultListableBeanFactory已经实现了此方法，如下方式：
```
        DefaultListableBeanFactory beanFactory = new DefaultListableBeanFactory();
        // 实例化基于 Properties 资源 BeanDefinitionReader
        PropertiesBeanDefinitionReader beanDefinitionReader = new PropertiesBeanDefinitionReader(beanFactory);
        String location = "META-INF/user.properties";
        // 基于 ClassPath 加载 properties 资源
        Resource resource = new ClassPathResource(location);
        // 指定字符编码 UTF-8
        EncodedResource encodedResource = new EncodedResource(resource, "UTF-8");
        int beanNumbers = beanDefinitionReader.loadBeanDefinitions(encodedResource);
        System.out.println("已加载 BeanDefinition 数量：" + beanNumbers);
        // 通过 Bean Id 和类型进行依赖查找
        User user = beanFactory.getBean("user", User.class);
        System.out.println(user);
```

堂堂一个DefaultListableBeanFactory，还需要注入到一个reader中？而不是reader注入到DefaultListableBeanFactory，DefaultListableBeanFactory依赖reader？你这DefaultListableBeanFactory已经实现了ConfigurableListableBeanFactory, BeanDefinitionRegistry接口，为什么不在实现个BeanDefinitionReader？

从实现上看，把reader于registry分开是非常有必要的，相当于reader是资源的生产，而registry是资源的消费。

如果DefaultListableBeanFactory中依赖reader，那么就需要提供一系列的针对各种不同的xml、properties的read的接口，这显然会明显的增加DefaultListableBeanFactory的接口数量，增加了DefaultListableBeanFactory的功能。而把自己作为一个registry方式给reader，自己只需要暴露一个registry接口，reader的工作完全不需要。

## BeanDefinition的注册
- 注册过程通过DefaultListableBeanFactory#registerBeanDefinition来实现，把BeanDefinition的通过beanName和value的方式保存在beanDefinitionMap（ConcurrentHashMap）中
- 为了记录注册的顺序，还保存了另外一个beanDefinitionNames的ArrayList，相当于LinkedConcurrentHashMap（没有这东西）。注意为了读取时候的安全性，修改这ArrayList的时候即使增加一个元素也是先把以前的拷贝一份，在拷贝的上增加后在把拷贝的给赋值给原来的引用。防止并发中的问题：
```
				synchronized (this.beanDefinitionMap) {
					this.beanDefinitionMap.put(beanName, beanDefinition);
					List<String> updatedDefinitions = new ArrayList<>(this.beanDefinitionNames.size() + 1);
					updatedDefinitions.addAll(this.beanDefinitionNames);
					updatedDefinitions.add(beanName);
					this.beanDefinitionNames = updatedDefinitions;
					removeManualSingletonName(beanName);
				}
```

## BeanDefinition的MR
- 由于在配置的过程中可以有parent的配置，对应的类的可能有父类，所以BeanDefinition需要把parent中的配置于当前的配置MR下
- 接口是在ConfigurableBeanFactory#getMergedBeanDefinition中，这个ConfigurableBeanFactory是个比较底层的接口，不光提供了一些可以config的接口，还提供了getMergedBeanDefinition这个比较基础的接口，获取到的所有的BeanDefinition都是需要经过MR的
- MR的过程是个递归的过程，把一个一般的GenericBeanDefinition最终转换为一个RootBeanDefinition，也就是说获取到BeanDefinition都是RootBeanDefinition

## Reslove bean class
- 这是个createBean过程中的一部分，AbstractBeanFactory#resolveBeanClass，入参为BeanDefinition与BeanName
- 解析class过程就是用AbstractBeanFactory的classloader加载BeanDefinition的class的过程

## 实例化前
- InstantiationAwareBeanPostProcessor，这PostProcessor中有个postProcessBeforeInstantiation方法，是在初始化之前（createBean）之前就调用，如果这个方法返回一个非空对象，就不在进行后续的实例化操作，直接getBean就返回了。确实是个非主流的操作
- 为什么叫PostProcessor？post不是后置的意思么？但这postProcessBeforeInstantiation又有个before，直接叫Processor不就可以了么

## 实例化
- 实例化使用的是反射方式进行实例化
- 通过构造器注入的时候，是根据类型进行注入的而不是beanName

## 实例化后
- InstantiationAwareBeanPostProcessor，与前面postProcessBeforeInstantiation对应的有个postProcessAfterInstantiation方法，这个方法中可以设置bean的value，而且如果返回false，那么后续就不在对bean的properties进行赋值。populateBean方法中在填充之前的一段逻辑如下,如果返回，之后的populate操作就不会执行了
```
		// Give any InstantiationAwareBeanPostProcessors the opportunity to modify the
		// state of the bean before properties are set. This can be used, for example,
		// to support styles of field injection.
		if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
			for (BeanPostProcessor bp : getBeanPostProcessors()) {
				if (bp instanceof InstantiationAwareBeanPostProcessor) {
					InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
					if (!ibp.postProcessAfterInstantiation(bw.getWrappedInstance(), beanName)) {
						return;
					}
				}
			}
		}
```

## 赋值之前
- 还是这个InstantiationAwareBeanPostProcessor接口，postProcessProperties这个接口会在applyPropertyValues之前可以修改替换PropertyValues。这名字就更奇怪了，post不是后置么，竟然是在之前操作，也没个before，代码实现如下，还是在populateBean方法内
```
for (BeanPostProcessor bp : getBeanPostProcessors()) {
	if (bp instanceof InstantiationAwareBeanPostProcessor) {
		InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
		PropertyValues pvsToUse = ibp.postProcessProperties(pvs, bw.getWrappedInstance(), beanName);
		if (pvsToUse == null) {
			if (filteredPds == null) {
				filteredPds = filterPropertyDescriptorsForDependencyCheck(bw, mbd.allowCaching);
			}
			pvsToUse = ibp.postProcessPropertyValues(pvs, filteredPds, bw.getWrappedInstance(), beanName);
			if (pvsToUse == null) {
				return;
			}
		}
		pvs = pvsToUse;
	}
}
```

## aware回调阶段、初始化前阶段
- 这接口回调也有点类似注入，可以理解为接口注入
- aware回调也有区别，一般的BeanFactory只能回调BeanNameAware、BeanClassLoaderAware、BeanFactoryAware三个，是在AbstractAutowireCapableBeanFactory#initializeBean开始的时候调用invokeAwareMethods触发的
- 其他的aware如EnvironmentAware、ApplicationContextAware是在AbstractApplicationContext#prepareBeanFactory过程中增加的beanFactory.addBeanPostProcessor(new ApplicationContextAwareProcessor(this));并且在bean初始化开始的阶段，在AbstractAutowireCapableBeanFactory#applyBeanPostProcessorsBeforeInitialization触发回调的
- 初始化前，还是在InstantiationAwareBeanPostProcessor，这个接口中，有postProcessBeforeInitialization接口，其实是从BeanPostProcessor这个最底层的接口继承来的。上面的applyBeanPostProcessorsBeforeInitialization就是在调用这postProcessBeforeInitialization接口：
```
	public Object applyBeanPostProcessorsBeforeInitialization(Object existingBean, String beanName)
			throws BeansException {

		Object result = existingBean;
		for (BeanPostProcessor processor : getBeanPostProcessors()) {
			Object current = processor.postProcessBeforeInitialization(result, beanName);
			if (current == null) {
				return result;
			}
			result = current;
		}
		return result;
	}
```