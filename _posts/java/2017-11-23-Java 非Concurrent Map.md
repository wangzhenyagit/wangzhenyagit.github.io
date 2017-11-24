---
layout: post
title: Java 非Concurrent Map
category: Java相关
tags: Java
---

## OverView ##
作为一个从C++转到Java的人，想到的STL标准的map不就一个么，底层红黑树，虽然search效率不如AVL树，但是insert和delete效率高，综合能力强，底层key还是排序的，按照key遍历顺序较快。STL也有hash map但是一直也不是标准。

Java中的map有三个，HashMap、TreeMap、LinkHashMap，java的集合框架的名字一般是 <Implementation-style><Interface>结构，前面是底层实现的数据结构，后面是接口，其他的分类可以参考Java8的集合框架的OverView-[collections-overview](https://docs.oracle.com/javase/8/docs/technotes/guides/collections/overview.html) ，博客表格显示丑直接移步[github](https://github.com/wangzhenyagit/wangzhenyagit.github.io/blob/master/_posts/java/2017-11-23-Java%20Map.md)。


|Interface|Hash Table|Resizable Array|Balanced Tree|Linked List|HashTable+LinkedList|
|----     | ---      |----           | ---         |----       | ---                |
|Set	  |HashSet   |               |	TreeSet	   |           | LinkedHashSet      |
|List	  |          | ArrayList     |             | LinkedList|	                |
|Deque	  | 	     |ArrayDeque	 |	           |LinkedDeque|	                |
|Map	  |HashMap	 |	             |TreeMap	   |           |LinkedHashMap       |


这篇的主题是左后一行，Map接口的三种实现。


## Java HashMap ##
### 概述 ###
直接引用下官网的说明[Class HashMap](https://docs.oracle.com/javase/8/docs/api/java/util/HashMap.html)：

> This implementation provides all of the optional map operations, and permits null values and the null key. (The HashMap class is roughly equivalent to Hashtable, except that it is unsynchronized and permits nulls.) This class makes no guarantees as to the order of the map; in particular, it does not guarantee that the order will remain constant over time.

说明了几个特点，允许null key，非线程安全，无序，而且顺序会改变。

> This implementation provides constant-time performance for the basic operations (get and put), assuming the hash function disperses the elements properly among the buckets. Iteration over collection views requires time proportional to the "capacity" of the HashMap instance (the number of buckets) plus its size (the number of key-value mappings). Thus, it's very important not to set the initial capacity too high (or the load factor too low) if iteration performance is important.

get和put操作复杂度O(1),对于Iteration操作，与HashMap的capacity有关的，capacity越大iteration越慢。

> An instance of HashMap has two parameters that affect its performance: initial capacity and load factor. The capacity is the number of buckets in the hash table, and the initial capacity is simply the capacity at the time the hash table is created. The load factor is a measure of how full the hash table is allowed to get before its capacity is automatically increased. When the number of entries in the hash table exceeds the product of the load factor and the current capacity, the hash table is rehashed (that is, internal data structures are rebuilt) so that the hash table has approximately twice the number of buckets.

由于是Hash计算，很可能冲突已经非常多了，但是hash的容量还没有满，所以需要提前对Hash进行扩容。

> As a general rule, the default load factor (.75) offers a good tradeoff between time and space costs. Higher values decrease the space overhead but increase the lookup cost (reflected in most of the operations of the HashMap class, including get and put). The expected number of entries in the map and its load factor should be taken into account when setting its initial capacity, so as to minimize the number of rehash operations. If the initial capacity is greater than the maximum number of entries divided by the load factor, no rehash operations will ever occur.

rehash的操作还是有消耗的，虽然做了优化。**最好的就是在设定hash的capacity的时候估计出大概会有多少个对象，避免出现rehash的操作。**

> If many mappings are to be stored in a HashMap instance, creating it with a sufficiently large capacity will allow the mappings to be stored more efficiently than letting it perform automatic rehashing as needed to grow the table. **Note that using many keys with the same hashCode() is a sure way to slow down performance of any hash table.** To ameliorate impact, when keys are Comparable, this class may use comparison order among keys to help break ties.

这里，如果有很多个hashCode一样的对象，或者冲突特别多的时候，Hash的效率会下降。

### JAVA8的优化 ###
> A HashMap stores data into multiple singly linked lists of entries (also called buckets or bins). All the lists are registered in an array of Entry (Entry<K,V>[] array) and the default capacity of this inner array is 16.

在JAVA8之前，是通过链表的方式来解决冲突问题，如下：

<img src="http://coding-geek.com/wp-content/uploads/2015/03/internal_storage_java_hashmap.jpg"/>

在JAVA8中，对这个结构进行了优化，当一个bucket中的node大于8时，会把linked list转成红黑树的结构，会有linked list和red black tree同时存在,参考[JAVA 8 improvements](http://coding-geek.com/how-does-a-hashmap-work-in-java/#JAVA_8_improvements)。

<img src="http://coding-geek.com/wp-content/uploads/2015/03/internal_storage_java8_hashmap.jpg" />

在使用HashMap时候需要注意：

- When using a HashMap, you need to find a hash function for your keys that spreads the keys into the most possible buckets. To do so, you need to avoid hash collisions. The String Object is a good key because of it has good hash function. Integers are also good because their hashcode is their own value.
- 初始的大小很重要，最好足够大，当然要考虑内存的浪费，不产生rehash。
- 大小要是2的指数

### 为什hash的capacity要是2的次幂 ###
实现hash的算法如下：
> This index of the bucket (linked list) is generated in 3 steps by the map:
> 
> - It first gets the hashcode of the key.
> - It rehashes the hashcode to prevent against a bad hashing function from the key that would put all data in the same index (bucket) of the inner array
> - It takes the rehashed hash hashcode and bit-masks it with the length (minus 1) of the array. This operation assures that the index can’t be greater than the size of the array. You can see it as a very computationally optimized modulo function.

```
// the "rehash" function in JAVA 8 that directly takes the key
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>16);
    }
// the function that returns the index from the rehashed hash
static int indexFor(int h, int length) {
    return h & (length-1);
}
```

在最后一步，为了不超出array的大小，不是对大小取余数，而是使用了效率更高的&操作，例如如果长度16，会与15也就是1111进行&操作，这样很快地把之前的hash值变成了0-15之间的array的index。如果大小设置成了17，与16也就是0001 0000进行与操作，那结果只有两个，16和0，所有的key都会在那个两个bucket上。

[JAVA 8 improvements](http://coding-geek.com/how-does-a-hashmap-work-in-java/#JAVA_8_improvements)中的解释方式，从代码反推，上面的解释还是不清楚。其实最重要的是为什么要这样写代码？用与操作这个步骤的目的其实是取余，而取余要进行除法运算，显然，效率比较低。而这里直接用与的方式代替了取余操作，效率高，但是正好这个与操作与取余结果一样，当然这里有个前提，就是与的值要是2的次幂，例如2,4,16等。其实映射到十进制特别好理解，如果一个十进制数要取余，而且要取余的数正好是10的次幂，例如对123456进行100取余，肯定不会傻傻的去计算了，直接保留后两位就可以了，前面的直接扔掉，对于如何保留后两位，二进制上就更好计算了，与操作么。这里的设计还是比较巧妙的。

长度是2的次幂，除了上面的这个取余运算效率高外，还有另外一个特点，当capacity不够要rehash的时候，速度快。因为要扩容rehash的时候，原来的hash值是有的，这个时候，只需要改下h & (length-1)中的length，而这个length是2的次幂，length-1从外观上，就是前面多了个“1”，例如从16,length-1为 "1111"，变成32，length-1变成了"11111"，那么hash值的index只会有两种变化，一是前面多了个0，而是前面多了个1，多了个0，相当于你在你的银行卡1的前面加0，从1块，变成01块，没啥变化，也就是说这个hash的index是没有变化的，而对于所有多了个1的hash值，相当于加了个数量级，如果是从16增加到32，新的hash的index变成了 oldindex + 16，直接计算出array的地址，移过去就好了。

抛开目前的实现，如果一个hash的大小扩大了一倍，那最理想的rehash的方式就是有一半（扩大了一倍）的hash均匀分布到新的空间（保持hash的效率），而又一半最好原地不动（减少内存操作和cpu周期）。而且判断是否要移动的方式要尽量快。目前的设计，恰恰满足了这几个条件，算法的力量。

### 参考 ###
[Java HashMap工作原理及实现](http://yikun.github.io/2015/04/01/Java-HashMap%E5%B7%A5%E4%BD%9C%E5%8E%9F%E7%90%86%E5%8F%8A%E5%AE%9E%E7%8E%B0/)


## Redis中的Hash ##

Redis的hash场景与JAVA的很像，Redis是单线程，有没有线程安全的处理机制，而且使用是在内存中。对比下实现的异同。

### Hash算法 ###
Java中的默认的Object类的hashCode()方法返回这个对象存储的内存地址的编号。而Redis使用MurmurHash2算法。

在Java中HashMap的key最好是个人为指定的int或string，最好是id号之类的有点意义的。而且尽量保证使用对象的同一个属性来生成hashCode()和equals()两个方法，例如id号。

### 根据Hash计算index的算法 ###
```
// 使用字典设置的哈希函数，计算键 key 的哈希值
hash = dict->type->hashFunction(key);

// 使用哈希表的 sizemask 属性和哈希值，计算出索引值
// 根据情况不同， ht[x] 可以是 ht[0] 或者 ht[1]
index = hash & dict->ht[x].sizemask;
```

与Java中的根据hash计算index算法一样，这sizemask也是（size-1）的二进制，参考[哈希算法](http://redisbook.com/preview/dict/hash_algorithm.html)。

### 如何处理hash collision ###
与Java8之前一样，使用链表解决冲突问题。

### 扩容rehash ###
Redis的扩容大小也是按照2的次幂倍扩展的，而且大小是2的次幂大小，应该也是与Java的方式一样一是在取余的时候速度快，而是在rehash的过程速度快。

不同的是Redis的hash的size的大小不仅可以扩容，而且可以缩小，另外，扩容的时候有渐进式扩容的方式，而且，能够进行渐进式扩容也得益于大小一直是2的次幂大小，像上面分析的，就的hash映射的时候只有两种情况，要么不动，要么向后移动，而不会出现后面的向前移动的情况。这样，只要一个记录扩容的index记录就可以了，这个index为原来大小的时候就是扩容完毕，具体过程可以参考[渐进式 rehash](http://redisbook.com/preview/dict/incremental_rehashing.html).

## Java LinkedHashMap ##
从接口上看，有Linked特性，又有Hash table的特性，JAVA8官网介绍如下：
> This implementation differs from HashMap in that it maintains a doubly-linked list running through all of its entries. This linked list defines the iteration ordering, which is normally the order in which keys were inserted into the map (insertion-order). Note that insertion order is not affected if a key is re-inserted into the map. (A key k is reinserted into a map m if m.put(k, v) is invoked when m.containsKey(k) would return true immediately prior to the invocation.)

可以看出与HashMap的不同，多维护了一个双向的链表，这样就可以保持插入的顺序，能够按照插入顺序访问，而且相同的key再次插入，不会对以前的顺序有影响。

## Java TreeMap ##
底层红黑树，get，put，remove的复杂度都是log(n)，当然最最明显特征就是内部按照key排序了。

虽然不如AVL树平衡，平衡的结果是导致树变化频繁，get效率高，但put效率不高。


## HashMap vs LinkedHashMap vs TreeMap ##
直接参考stackoverflow上的一个答案：
[Difference between HashMap, LinkedHashMap and TreeMap](https://stackoverflow.com/a/17708526)

所有的不同原因可以说是底层的数据结构导致的，但是根本的还是因为不同场景有不同的需求，如果只是get和put，没有什么按照插入顺序、按照key的顺序遍历的需求，那就Hash就是第一选择。还有三种中TreeMap是不允许有null值的，还要调用key的equals方法比较大小呢，其他两种，null是可以的。

之所以对这个是否允许空值说下，是因为有的结构不那么容易得出是否允许null值，比如ConcurrentHashMap这，底层实现很复杂，先上结论，这结构不允许null值。原因后面分析。