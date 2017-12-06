---
layout: post
title: Java Volitale
category: Java相关
tags: Java
---

### 为什么要使用Volatile ###
> Volatile变量修饰符如果使用恰当的话，它比synchronized的使用和执行成本会更低，因为它不会引起线程上下文的切换和调度。

### Volatile的实现原理 ###

> 在x86处理器下通过工具获取JIT编译器生成的汇编指令来看看对Volatile进行写操作CPU会做什么事情。 
Java代码：instance = new Singleton();//instance是volatile变量  


> 对应汇编代码两条：  
movb $0x0,0x1104800(%esi);  
lock addl $0x0,(%esp);  


> 有volatile变量修饰的共享变量进行写操作的时候会多第二行汇编代码，通过查IA-32架构软件开发者手册可知，lock前缀的指令在多核处理器下会引发了两件事情。
> 
- 将当前处理器缓存行的数据会写回到系统内存。 
- 这个写回内存的操作会引起在其他CPU里缓存了该内存地址的数据无效。 
 
对于这关键字，有个通常的说法“volatile仅仅用来保证该变量对所有线程的可见性，但不保证原子性”。

volatile并不适合getAndSet的操作，如典型的i++操作。对于一个volatile的i++操作，分为以下个步骤：

1. 从内存加载到cpu的寄存器
2. 寄存器对i进行++操作
3. 寄存器值写回到内存
4. 执行cpu的内存屏障（Memory Barrier），防止cpu进行优化不写回，强制执行第三步骤的操作,使其他的cpu寄存器中的i的值无效，cpu在操作i必须重新从内存加载一次

没有原子性是因为，上面的几个个步骤，并不是一个原子操作，多线程并发的i++会有问题。但是只要是有一个线程更新了值，那么其他线程读到的值，都是一致的，这就叫做“可见性”。

如果不是volatile变量，多线程对数据操作就没有可见性了么？那么一种情况，对于一个char（一个字节）进行++，一个线程进行++后，其他线程读到的值会有延后么？通过下面代码测试下。

```
import java.util.concurrent.Semaphore;

public class VolatileTest {
	static byte count = 0;
	final static Semaphore available = new Semaphore(0);
	
	public static void main(String[] args) {
		
		new Thread(){
			public void run(){
				while(true){
					count++;
					available.release();
					System.out.println("change count : " + count);
					try {
						Thread.sleep(100);
					} catch (InterruptedException e) {
						e.printStackTrace();
					}
				}

			}
		}.start();
		
		new Thread(){
			public void run(){
				while(true){
					try {
						available.acquire();
					} catch (InterruptedException e) {
						// TODO Auto-generated catch block
						e.printStackTrace();
					}
					System.out.println("get count : " + count);
				}
				
				
			}
		}.start();
		
		System.out.println("Main End.");
	}
}
```

额，在我机器上，这count第二个线程读取的时候没有“延迟”的问题，都是更新后的值，这“可见性”的验证，待研究。

如果volatile有可见性的特性，那么锁也应该有可见性这个特性。这么看来，锁其实有两个维度互斥和可见，而volatile只有一个可见的维度，所以，可以一些情况下把volatile看作一个轻量级的锁。

### 适用的场景 ###
典型场景就是写少，但读非常多。这样读的时候其实和普通变量很像，直接是从cpu的寄存器中读取的，也不用获取锁。因为获取锁也是比较消耗资源的，典型的如果synchronize关键字同步（如果是从偏向锁、轻量级锁）一路升到重量级锁，获取锁的时候，要进行锁的竞争，对象头中的锁的指针要不断的修改，与没有锁的volatile相比，开销很大。

在高效的结构如currentHashMap和Disruptor中都有使用volatile变量。

参考：
[为什么volatile不能保证原子性而Atomic可以？](http://www.cnblogs.com/Mainz/p/3556430.html)

