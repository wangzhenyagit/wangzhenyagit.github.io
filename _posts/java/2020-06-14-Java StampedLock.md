---
layout: post
title: Java StampedLock
category: Java相关
tags: spring
---

## StampedLock
> A capability-based lock with three modes for controlling read/write access. The state of a StampedLock consists of a version and mode. Lock acquisition methods return a stamp that represents and controls access with respect to a lock state; "try" versions of these methods may instead return the special value zero to represent failure to acquire access. Lock release and conversion methods require stamps as arguments, and fail if they do not match the state of the lock.

这stamp有票据的意思，邮票的意思，与timestamp中的stamp语义有些类似，代表一个过去的曾经的状态。

官方文档上的例子：

```
 class Point {
   private double x, y;
   private final StampedLock sl = new StampedLock();

   void move(double deltaX, double deltaY) { // an exclusively locked method
     long stamp = sl.writeLock();
     try {
       x += deltaX;
       y += deltaY;
     } finally {
       sl.unlockWrite(stamp);
     }
   }

   double distanceFromOrigin() { // A read-only method
     long stamp = sl.tryOptimisticRead();
     double currentX = x, currentY = y;
     if (!sl.validate(stamp)) {
        stamp = sl.readLock();
        try {
          currentX = x;
          currentY = y;
        } finally {
           sl.unlockRead(stamp);
        }
     }
     return Math.sqrt(currentX * currentX + currentY * currentY);
   }

   void moveIfAtOrigin(double newX, double newY) { // upgrade
     // Could instead start with optimistic, not read mode
     long stamp = sl.readLock();
     try {
       while (x == 0.0 && y == 0.0) {
         long ws = sl.tryConvertToWriteLock(stamp);
         if (ws != 0L) {
           stamp = ws;
           x = newX;
           y = newY;
           break;
         }
         else {
           sl.unlockRead(stamp);
           stamp = sl.writeLock();
         }
       }
     } finally {
       sl.unlock(stamp);
     }
   }
 }
```

还有一句类似于警告的"its use is inherently fragile"：
```
 Method tryOptimisticRead() returns a non-zero stamp only if the lock is not currently held in write mode. Method validate(long) returns true if the lock has not been acquired in write mode since obtaining a given stamp. This mode can be thought of as an extremely weak version of a read-lock, that can be broken by a writer at any time. The use of optimistic mode for short read-only code segments often reduces contention and improves throughput. However, its use is inherently fragile. Optimistic read sections should only read fields and hold them in local variables for later use after validation. Fields read while in optimistic mode may be wildly inconsistent, so usage applies only when you are familiar enough with data representations to check consistency and/or repeatedly invoke method validate(). For example, such steps are typically required when first reading an object or array reference, and then accessing one of its fields, elements or methods.
```

者乐观锁是脆弱的，对demo的代码，有几个疑问：

### 对于double类型的x会有写了一半然后读取到的问题么？
理论上会，但是按照demo给的模板是没有问题的，如下代码：
```
   double distanceFromOrigin() { // A read-only method
     long stamp = sl.tryOptimisticRead();
     double currentX = x, currentY = y;
     if (!sl.validate(stamp)) {
		xxx
	}
	return Math.sqrt(currentX * currentX + currentY * currentY);
```
获取乐观读，然后赋值，在检查，这个时候，理论上x、y会有只更新了x但没有更新y的问题，甚至有只更新了x中的8个字节中的4个字节的问题，但是由于下面的validate是能检测出来这种变化的（保证了可见性）。

而且，这里是需要copy一份的，如果不copy，那return Math.sqrt(x * x + y * y);就会出现上面的场景，因为多个步骤是不能保证同步的。

所以，使用的过程中要严格使用上述的模板，validate，不可少。关键点：获取乐观锁->copy需要用到的资源->检查是否版本已修改->使用copy的值进行逻辑，或者如已修改获取悲观锁再次获取值。

### 不可重入的
不是可重入的。从源码上看，无论是写、悲观还是乐观读，都没有记录当前锁的exclusiveOwnerThread是哪个，也就是说，使用锁的时候，对于锁来说线程的状态是透明的，已经持有互斥的W的锁的线程和其他线程是一样的。但是，重入，也分那种重入，对于可重入读写锁而言，下面的操作在同一线程中是不会阻塞的：

```
    public static void main(String[] args) {
        ReentrantReadWriteLock rrw = new ReentrantReadWriteLock();
        rrw.writeLock().lock();
        rrw.readLock().lock();
        rrw.writeLock().lock();
        rrw.readLock().lock();
```

在可重入读写锁实现的时候，都有去检查当前线程是否为持有互斥写锁的操作，是的话获取锁的时候就会通过。而对于StampedLock，由于是不区分是否为当前线程所以重入性没有考虑，但是本身从锁的自身的特性上，读读操作都是可重入的，不管是悲观读+悲观读的重入还是乐观读的重入，而悲观读+写或者写+写都是不可重入的。
```
    public static void main(String[] args) {
        StampedLock sl = new StampedLock();
        long stampRead1 = sl.readLock();
        System.out.println("read lock " + stampRead1);
        long stampRead2 = sl.readLock();
        System.out.println("read lock " + stampRead2);

        System.out.println("optimistic read lock " + sl.tryOptimisticRead());
        System.out.println("optimistic read lock " + sl.tryOptimisticRead());

        // 如不释放，后续写锁会阻塞
        sl.unlockRead(stampRead1);
        sl.unlockRead(stampRead2);

        System.out.println("optimistic read lock " + sl.tryOptimisticRead());

        long stamp = sl.writeLock();
        System.out.println("write lock " + stamp);

        // 如不释放后续写锁会阻塞
        sl.unlock(stamp);
        stamp = sl.writeLock();
        System.out.println("write lock " + stamp);
    }
```
输出：
```
read lock 257
read lock 258
optimistic read lock 256
optimistic read lock 256
optimistic read lock 256
write lock 384
write lock 640
```


## 参考
[doc](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/locks/StampedLock.html#tryConvertToWriteLock-long-)