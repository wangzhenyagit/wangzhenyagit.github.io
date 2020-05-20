---
layout: post
title: Java Integer
category: Java相关
tags: Java
---

## 线程安全

原始数据类型并不是线程安全的，所以才有了AtomicInteger、AtomicLong这样的类。

## 缓存

integer的valueOf是有缓存的，这个值默认缓存是 -128 到 127 之间。最大值是可以由jvm参数进行配置的，代码如下：
```
 private static class IntegerCache {
        static final int low = -128;
        static final int high;
        static final Integer cache[];

        static {
            // high value may be configured by property
            int h = 127;
            String integerCacheHighPropValue =
                sun.misc.VM.getSavedProperty("java.lang.Integer.IntegerCache.high");
            if (integerCacheHighPropValue != null) {
                try {
                    int i = parseInt(integerCacheHighPropValue);
                    i = Math.max(i, 127);
                    // Maximum array size is Integer.MAX_VALUE
                    h = Math.min(i, Integer.MAX_VALUE - (-low) -1);
                } catch( NumberFormatException nfe) {
                    // If the property cannot be parsed into an int, ignore it.
                }
            }
            high = h;

            cache = new Integer[(high - low) + 1];
            int j = low;
            for(int k = 0; k < cache.length; k++)
                cache[k] = new Integer(j++);

            // range [-128, 127] must be interned (JLS7 5.1.7)
            assert IntegerCache.high >= 127;
        }

        private IntegerCache() {}
    }

	
	// 获取的时候直接从数组中取
    public static Integer valueOf(int i) {
        if (i >= IntegerCache.low && i <= IntegerCache.high)
            return IntegerCache.cache[i + (-IntegerCache.low)];
        return new Integer(i);
    }

```

PS: 虽然Integer中用数组进行缓存，访问效率比较高，但是与基础的数据的类型相比，需要通过引用进行二次访问，而且对应的对象存储也是不连续的，无法有效利用cpu的缓存。但是引用还是连续的，引用能利用缓存。

Map的开放地址法中的线性探测在散列，比链地址法更能有效的利用cpu缓存。

## constant

与String一样，Integer也是constant的，第一个出发点也是多线程的安全考虑。

第二个，这个设计的出发点是尽量无感知的与int一样使用。自动装箱与自动拆箱也是这个思路，如果不是constant的，那么使用的时候就与int有比较大的区别了，这个就很容易陷入坑里面。

## 占用空间

显然Integer的空间是大于int的，int是固定四个字节的。

而对象有固定的字节开销，对象一般是对象头、内容、对齐填充。

对象头32和64位不同，32bit和64bit的，如哈希码（HashCode）、GC分代年龄、锁状态标志、线程持有的锁、偏向线程ID、偏向时间戳等，还有对象的类型指针。

第二部分是实例数据。

第三部分是为了访问高效，把对象填充到8字节的整数倍数。

## 参考
[int和Integer的区别](https://time.geekbang.org/column/article/7514)