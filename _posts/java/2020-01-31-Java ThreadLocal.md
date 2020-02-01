---
layout: post
title: Java ThreadLocal
category: Java相关
tags: Java
---

## 既然ThreadLocal是以弱引用方式存储的，那么会不会设置的值还没用就被GC了？

先说结论，不会。看了下源码，看似简单的代码，很多弯弯绕。以下代码引用为jdk1.8。  

这个ThreadLocal的类所在的包为java.lang，与Thread类在同一包下，也就是有类似c++友元类的特性，ThreadLocal类能直接访问Thread类的私有成员。而且ThreadLocal类设计的时候使用了这特性，在Thread类中，有个私有成员为：

```
/* ThreadLocal values pertaining to this thread. This map is maintained by the ThreadLocal class. */
ThreadLocal.ThreadLocalMap threadLocals = null;
```

注释上也说了，This map is maintained by the ThreadLocal class，而且还定义在ThreadLocal类中。看下ThreadLoacl最常用的set方法：

```
    public void set(T value) {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
    }
```

通过ThreadLocal对象的set方法，结果把ThreadLocal对象自己当做key，放进了ThreadLoalMap中。也就是说，ThreadLocal中什么数据也没保存，数据存放在了Thread的ThreadLocalMap中，而且自己就是key。设计的实在巧妙，一是在使用上非常简单，new个ThreadLocal，然后set就可以了，二是在存储的时候直接使用对象自己作为key，而不需要用户在刻意去生成一个唯一的key，然后在使用的时候用这个定义的key去get到value。

分析下开始的问题，这ThreadLocalMap中存放的是Entry，继承自WeakReference，然后多加了个value的成员，也就是说key是个weakReference，在没有强引用的情况下是会被GC的，但是有个前提就是“没有其他的强引用”，而在使用ThreadLocal时候，都是ThreadLocal threaLocal = new ThreadLocal();这种方式初始化的，而这个threadLocal就是那个强引用，在使用之前，只要没有去threadLocal = null就不会被GC掉。
```
/**
 * The entries in this hash map extend WeakReference, using
 * its main ref field as the key (which is always a
 * ThreadLocal object).  Note that null keys (i.e. entry.get()
 * == null) mean that the key is no longer referenced, so the
 * entry can be expunged from table.  Such entries are referred to
 * as "stale entries" in the code that follows.
 */
static class Entry extends WeakReference<ThreadLocal<?>> {
    /** The value associated with this ThreadLocal. */
    Object value;

    Entry(ThreadLocal<?> k, Object v) {
        super(k);
        value = v;
    }
}
```

看下找到的一个图，清楚表示了上述关系：
![](https://user-gold-cdn.xitu.io/2018/4/3/162896ab1a1d1e2e?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

图中的ThreadLocal Ref就是上文中的threadLocal，也就是强引用，放到Entry中的时候整个Entry是个ThreadLocal的弱引用，还包括了一个对value的强引用。


## 使用后直接threadLocal = null，会有内存泄漏么？

大概率会，正确的方式是threadLocal.remove()。为什么说会大概率会呢？看下ThreadLocalMap的set的方法，后面有个方法cleanSomeSlots，代码如下：

```
private void set(ThreadLocal<?> key, Object value) {

    // We don't use a fast path as with get() because it is at
    // least as common to use set() to create new entries as
    // it is to replace existing ones, in which case, a fast
    // path would fail more often than not.

    Entry[] tab = table;
    int len = tab.length;
    int i = key.threadLocalHashCode & (len-1);

    for (Entry e = tab[i];
         e != null;
         e = tab[i = nextIndex(i, len)]) {
        ThreadLocal<?> k = e.get();

        if (k == key) {
            e.value = value;
            return;
        }

        if (k == null) {
            replaceStaleEntry(key, value, i);
            return;
        }
    }

    tab[i] = new Entry(key, value);
    int sz = ++size;
    if (!cleanSomeSlots(i, sz) && sz >= threshold)
        rehash();
}

/**
 * Heuristically scan some cells looking for stale entries.
 * This is invoked when either a new element is added, or
 * another stale one has been expunged. It performs a
 * logarithmic number of scans, as a balance between no
 * scanning (fast but retains garbage) and a number of scans
 * proportional to number of elements, that would find all
 * garbage but would cause some insertions to take O(n) time.
 */
private boolean cleanSomeSlots(int i, int n) {
    boolean removed = false;
    Entry[] tab = table;
    int len = tab.length;
    do {
        i = nextIndex(i, len);
        Entry e = tab[i];
        if (e != null && e.get() == null) {
            n = len;
            removed = true;
            i = expungeStaleEntry(i);
        }
    } while ( (n >>>= 1) != 0);
    return removed;
}
```
首先这个方法的名字cleanSomeSlots，而不是cleanSlots，这个some还真实some，看下注释，是个balance的方式，如果都扫描下性能差，如果不扫描garbage太多，也就是泄露的比较多。remove会直接调用expungeStaleEntry这个方法把Entry中的key和value都设置为null，这样就可以让GC掉value了。如果key已经为null了（threadLocal外部强引用为null，并且gc过了key（弱引用）），那么就直接把value设置为null。

```
private int expungeStaleEntry(int staleSlot) {
    Entry[] tab = table;
    int len = tab.length;

    // expunge entry at staleSlot
    tab[staleSlot].value = null;
    tab[staleSlot] = null;
    size--;

    // Rehash until we encounter null
    Entry e;
    int i;
    for (i = nextIndex(staleSlot, len);
         (e = tab[i]) != null;
         i = nextIndex(i, len)) {
        ThreadLocal<?> k = e.get();
        if (k == null) {
            e.value = null;
            tab[i] = null;
            size--;
        } else {
            int h = k.threadLocalHashCode & (len - 1);
            if (h != i) {
                tab[i] = null;

                // Unlike Knuth 6.4 Algorithm R, we must scan until
                // null because multiple entries could have been stale.
                while (tab[h] != null)
                    h = nextIndex(h, len);
                tab[h] = e;
            }
        }
    }
    return i;
}
```

## 为什么Entry的key需要为WeakReference？
如果entry的key不是WeakReference，在使用线程池的情况下，如果一个流程中的在set后没有去调用remove（需要开发个新的remove方法），即使调用了threadLocal = null也会导致内存泄漏。由于此ThreadLocalMap属于线程，而ThreadLocalMap中的Entry也是有个强引用的key（ThreadLocal），那么第这个new出来的ThreadLocal就会内存泄漏了，而且对应的value也会泄漏。如果为WeakReference，在外部的强引用没有后，gc就会首先把key（ThreadLocal）给回收掉，然后在set、get操作下触发cleanSomeSlots，解决大部分的内存泄漏问题。

## What is a canonicalizing mapping, why would I use one and how do weak references help?

java jdk对于weakreference的说明如下：
>  Weak reference objects, which do not prevent their referents from being
>  made finalizable, finalized, and then reclaimed. Weak references are
>  most often used to implement canonicalizing mappings. 

什么是canonicalizing mapping？google的翻译是“规范化”？wiki上的一个词条[CanonicalizedMapping](https://wiki.c2.com/?CanonicalizedMapping)

A "canonicalized" mapping is where you keep one instance of the object in question in memory and all others look up that particular instance via pointers or somesuch mechanism. This is where weaks references can help.
The short answer is that WeakReference objects can be used to create pointers to objects in your system while still allowing those objects to be reclaimed by the garbage-collector once they pass out of scope. For example if I had code like this:
```
 class Registry {
     private Set registeredObjects = new HashSet();

     public void register(Object object) {
         registeredObjects.add( object );
     }
 }
 ```

Any object I register will never be reclaimed by the GC because there is a reference to it stored in the set of registeredObjects. On the other hand if I do this:
```
 class Registry {
     private Set registeredObjects = new HashSet();

     public void register(Object object) {
         registeredObjects.add( new WeakReference(object) );
     }
 }
 ```

Then when the GC wants to reclaim the objects in the Set it will be able to do so.
You can use this technique for caching, cataloguing, etc. See below for references to much more in-depth discussions of GC and caching.

强行理解下，在使用map的场景下，key使用weakReference类型，才是“规范化”的。不使用虽然问题可以避免（手动去清理map）但使用了大多数场景下好像也没有太多的副作用。所以，如果下次使用Map，对于key可以先考虑下，是否需要使用weakReference。

就上上面的例子，订阅者的列表使用map存放，因为map中本来就有个对与注册对象的强引用，如果只是简单的，new出来的外部对象也有个强引用，而通常来说，java开发者简单的把外部的对象给置null就会认为gc会收集此对象，但是因为有map中的强引用，会一直不能gc掉。所以有了者weakReference的使用场景。  

和ThreadLocal的场景很像，ThreadLocal中这样使用可以避免一定程度的内存泄漏，把废弃的ThreadLocal给赋值null，虽然并不是一个好的实践（应该调用remove方法），但也可以减少一定程度的内存泄漏。  

## 为什么ThreadLocal使用容易出问题？
个人感觉，封装的太“好”了，大家只知道set和get，很多不知道需要调用个remove。而且使用完一个对象去调用remove方法，很非主流。如果不是看了源码，鬼知道去调用remove。如果线程池的场景，如果使用完不去remove，内存泄漏非常容易，是个明显的java的坑。主要是隐藏了信息，线程可能是复用的，还有就是ThreadLocal存储在了map中。

## set，get的时候为什么没有加锁？为什么没有直接使用HashMap？
不加锁是因为调度的问题，不会出现一个线程的时间片同时被执行的情况。

为什么没有使用HashMap？set的时候是在自己计算table中的offset，而不是直接用的HashMap。  

Hash冲突解决的方式主要大概分两种，Separate chaining和Open addressing，翻译为单独链表法与开放地址法。

HashMap的Separate chaining with linked lists方式特点是数据结构简单，对于存储数量较多的数据时候，仍能有一定的性能保证。

> Chained hash tables with linked lists are popular because they require only basic data structures with simple algorithms, and can use simple hash functions that are unsuitable for other methods.
> 
> The cost of a table operation is that of scanning the entries of the selected bucket for the desired key. If the distribution of keys is sufficiently uniform, the average cost of a lookup depends only on the average number of keys per bucket—that is, it is roughly proportional to the load factor.
> 
> For this reason, chained hash tables remain effective even when the number of table entries n is much higher than the number of slots. For example, a chained hash table with 1000 slots and 10,000 stored keys (load factor 10) is five to ten times slower than a 10,000-slot table (load factor 1); but still 1000 times faster than a plain sequential list.

缺点就是，因为为链表，有个next的空间存储浪费，且由于list的插入、删除导致结构变化缓存失效的很快。

> Chained hash tables also inherit the disadvantages of linked lists. When storing small keys and values, the space overhead of the next pointer in each entry record can be significant. An additional disadvantage is that traversing a linked list has poor cache performance, making the processor cache ineffective.

而ThreadLocalMap的方式用的开放地址中的Linear probing（线性探测）方法，最大优势就是容易缓存，存储的是个连续的空间，有好的访问局部性（locality of reference），而且越是ThreadLocalMap的size越小，访问局部性特性就越明显，而ThreadLocal设计初衷并不是为了数据缓存较大规模的数据存储使用，只是解决并发中变量访问的问题。so，线性探测的方式主要是为了性能，而不是减少那些存储空间（线程数不会很多，ThreadLocalMap也不会很多）。

> Linear probing provides good locality of reference, which causes it to require few uncached memory accesses per operation. Because of this, for low to moderate load factors, it can provide very high performance. 

还有为什么叫opend addressing？
> The name "open addressing" refers to the fact that the location ("address") of the item is not determined by its hash value. 

通俗理解，地址并不是绝对地会被哪个item占用的，而是open的。像单独链表法中，一个item每次都会占用同样的bucket。

## 参考
[hash table](https://en.wikipedia.org/wiki/Hash_table)  
[Linear_probing](https://en.wikipedia.org/wiki/Linear_probing)
