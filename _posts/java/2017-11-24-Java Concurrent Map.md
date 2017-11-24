---
layout: post
title: Java Concurrent Map
category: Java相关
tags: Java
---

实现了ConcurrentMap接口的Map在Java中有两个：ConcurrentHashMap, ConcurrentSkipListMap。按照非并发的Map的经验，这两者的区别应该还是在是否有order上，正是的，这ConcurrentSkipListMap可以按照key顺序访问。

## Skip list 与 ConcurrentSkipListMap##
参考wiki：
> a skip list is a data structure that allows fast search within an ordered sequence of elements. 

<img src="https://upload.wikimedia.org/wikipedia/commons/thumb/2/2c/Skip_list_add_element-en.gif/400px-Skip_list_add_element-en.gif" />

这个skip list的结构的get、put也是O(log(n))的操作，那这结构与red-blcak tree比有哪些优势呢？

从使用的角度能够推断，同样的对key顺序排放的TreeMap，但是变成支持并发的后，底层结构变成了Skip list，那可以推断，Skip list并发情况在表现比red-black tree要好。
内存占用上

并发度上

cache上

范围查询上




