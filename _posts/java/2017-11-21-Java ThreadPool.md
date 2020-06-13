---
layout: post
title: Java ThreadPool
category: Java相关
tags: Java
---

## Executors
- 内置的newCachedThreadPool()使用场景是处理大量短时间的工作任务的线程，试图对线程缓存，内部使用SynchronousQueue为工作队列。实际中用到的不多，这线程池比较危险，因为线程池的最大的数目的max_value,弄不好，线程池如果不是短时间，线程数目飙升会严重影响系统。
- newFixedThreadPool()是无界的，其实也是有危险的。
- newSingleThreadExecutor（）这个能够方便模拟协程的方式，Future future = newSingleThreadExecutor().submit(task)。
- 对于线程池的状态ctl有个优化，ctl 变量被赋予了双重角色，通过高低位的不同，既表示线程池状态，又表示工作线程数目。
