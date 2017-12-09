---
layout: post
title: Singleton pattern
category: Java相关
tags: Java
---

单例模式实现方式比较多，有饿汉模式、懒汉模式（DCL）、枚举、静态内部类，基本上这四种。这懒汉模式，也没有特别的定义，一般是指DCL双重锁检查实现的方式。这懒汉模式，其实是指的“Lazy initialization”的方式，

### 饿汉模式 ###
如果是比较小的项目，或者没有延迟加载的需求，饿汉模式，自己也经常用。一直在搜索这“饿汉模式”中“饿汉”的由来，google了半天也没找到，感觉这像中国人起的名字额，参考[Singleton Design Pattern in Java](https://howtodoinjava.com/design-patterns/creational/singleton-design-pattern-in-java/)中，称为“Eager initialization”，这eger的翻译意思是“(of a person) wanting to do or have something very much.”，看前面的主语是个person，所以才翻译成饿汉，也隐含此模式下，会提前初始化。提前的意思是在类装载的时候会初始化。

类的声明周期又加载、连接、初始化、使用、卸载四个过程，其中初始化阶，对于初始化阶段有5个可以触发的方式，最常见的就是new一个对象、使用类的静态方法、读取或者设置类的静态字段等。初始阶段过程直接引用《深入理解Java虚拟机》中的描述：

> 初始化阶段是执行类构造器<clinit>()方法的过程.<clinit>()方法是由编译器自动收集类中的所有类变量的赋值动作和静态语句块static{}中的语句合并产生的

其实也就是对类的变量和静态变量进行赋值（包括静态变量）和静态语句的执行。

> 虚拟机会保证一个类的<clinit>()方法在多线程环境中被正确的加锁、同步，如果多个线程同时去初始化一个类，那么只会有一个线程去执行这个类的<clinit>()方法，其他线程都需要阻塞等待，直到活动线程执行<clinit>()方法完毕。

另外，虚拟机调用的这class init方法(clinit)是线程安全的，有专门的锁，保证类的加载初始化不乱。

等等，那为什么说这是模式是Eager初始化呢？因为单例中的lazy初始化，说的是调用getInstace的时候在初始化，但是上面说了，触发初始化有5个方式呢，可能不是调用getInstance方法，而触发了类的初始化，比如，下面的静态方法getDesc方法，不知道在哪里调用了下，就会导致类的初始化了。

```
public class EagerSingleton {
    private static volatile EagerSingleton instance = new EagerSingleton();
    
    // private constructor
    private EagerSingleton() {
		System.out.println("EagerSingleton");
    }
 
    public static EagerSingleton getInstance() {
        return instance;
    }

	private static String name = "Test Singleton.";
    public static String getDesc() {
        return name;
    }
    
	public static void main(String[] args) {
		EagerSingleton.getDesc();
	}

}
```

### 懒汉模式 ###
有两种实现方式，一种的DCL，一种是静态内部类的方式，DCL的方式如下：
```
public final class LazySingleton {
    private static volatile LazySingleton instance = null;
 
    // private constructor
    private LazySingleton() {
    }
 
    public static LazySingleton getInstance() {
        if (instance == null) {
            synchronized (LazySingleton.class) {
                instance = new LazySingleton();
            }
        }
        return instance;
    }
}
```

这种方式要在Java5后对volatile完善后才可以，为啥是要加这个volatile？cpu为了提高速度，对指令有一定的优化，如：

```
int a=1;
int b=2;
int c=a+b;
```

上述三行代码，CPU执行的时候，可能b=2是先优于a=1执行的，因为对第三行的结果不会有什么影响。额，那么，instance = new LazySingleton();如果这instance不是volatile的，此赋值步骤如下：

1. 分配一块对象内存空间
2. 对于对象进行初始化
3. 把指针指向instance

如果是单个CPU上，那么步骤可能是1->3->2,因为即使顺序改变了，在单个CPU对后续操作也不会有问题。但是在多线程下，如果步骤是1->3->2，其他线程可能会拿到一个没有初始后好的对象，从而引起问题，加上了volatile，有额外的“不允许指令重排”的光环，就不会有这问题了。

另外一种实现方式，叫做“Initialization-on-demand holder idiom”，利用静态内部类的方式实现，如下：

```
public class InnerClassSingleton {
	private static int version = 1;
	
	public static int getVersion() {
		return version;
	}
	
	private InnerClassSingleton() {
		System.out.println("Init InnerClassSingleton.");
	}
	
	private static class LazyHolder{
		static final InnerClassSingleton instance = new InnerClassSingleton();
	}
	
	public static InnerClassSingleton getInstance() {
		return LazyHolder.instance;
	}
	
	public static void main(String[] args) {
		InnerClassSingleton.getVersion();                   // 运行此行代码不会进行初始化
		System.out.println("Mark");
		InnerClassSingleton.getInstance().getVersion();     // 这里才会初始化
	}

}
```

会输出:

```
Mark
Init InnerClassSingleton.
```


### 枚举方式 ###
枚举的方式，是后出现的，写法也是最简洁，是 Effective Java中推荐的格式，代码如下：
```
public enum EnumSingleton {
	INSTANCE;
	
	public void foo() {
	}

}
```

上面的是单利模式的最佳实践，那么，这个方式是lazy加载的么？参考[Singleton via enum way is lazy initialized?](https://stackoverflow.com/questions/16771373/singleton-via-enum-way-is-lazy-initialized)。

是懒加载的，但是不是单例中想要的懒加载，与饿汉模式的很像，测试代码如下，在调用getVersion方法的时候，枚举中有几个枚举值，就会初始化几次，下面的代码会打印两次"enum init."

```
public enum EnumSingleton {
	INSTANCE1, INSTANCE2;
	
	private static int version = 1;
	
	private EnumSingleton() {
		System.out.println("enum init.");
	}
	
	public static int getVersion() {
		return version;
	}
	
	public static void main(String[] args) {
		EnumSingleton.getVersion();
	}

}
```

### 总结 ###
实际项目中使用，既然都是最佳实践了，那就用枚举方式把，虽然有一些方式会触发提前的初始化，但是，这些方式也是理论上的，实际中很少这样做，类的初始化的触发有六个场景如下：

> - A new instance of a class is created (in bytecodes, the execution of a new instruction. Alternatively, via implicit creation, reflection, cloning, or deserialization.)
> - The invocation of a static method declared by a class (in bytecodes, the execution of an invokestatic instruction)
> - The use or assignment of a static field declared by a class or interface, except for static fields that are final and initialized by a compile-time constant expression (in bytecodes, the execution of a getstatic or putstatic instruction)
> - The invocation of certain reflective methods in the Java API, such as methods in class Class or in classes in the java.lang.reflect package
> - The initialization of a subclass of a class (Initialization of a class requires prior initialization of its superclass.)
> - The designation of a class as the initial class (with the main()< method) when a Java virtual machine starts up


## 参考 ##
[Java：单例模式的七种写法](http://www.blogjava.net/kenzhh/archive/2016/03/28/357824.html)  
[双重检查锁定与延迟初始化](http://www.infoq.com/cn/articles/double-checked-locking-with-delay-initialization)  
[Singleton Design Pattern in Java](https://howtodoinjava.com/design-patterns/creational/singleton-design-pattern-in-java/)