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

## 为什么Entry的key需要为WeakReferer？
如果entry的key不是WeakReference，在使用线程池的情况下，如果一个流程中的在set后没有去调用remove（需要开发个新的remove方法），即使调用了threadLocal = null也会导致内存泄漏。由于此ThreadLocalMap属于线程，而ThreadLocalMap中的Entry也是有个强引用的key（ThreadLocal），那么第这个new出来的ThreadLocal就会内存泄漏了，而且对应的value也会泄漏。如果为WeakReference，在外部的强引用没有后，gc就会首先把key（ThreadLocal）给回收掉，然后在set、get操作下触发cleanSomeSlots，解决大部分的内存泄漏问题。


