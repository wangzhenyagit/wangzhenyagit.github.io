---
layout: post
title: ES（五）Why Skip List
category: 存储系统
tags: es
---

## Why Skip List ? ##

**为了支持全文搜索，Lucene中使用了倒排索引，但是倒排索引也有很多分词后的词(Term)，而且这些词的数目可能是非常大的，那如何对这些倒排索引的词进行快速的检索呢？很容易想到的就是在对这些词建个索引么，直接想到的就是，量大的化内存放不下，那就像Mysql一样，B+Tree数据结构搞个索引。但是在luence中，使用的是skip-list，why？**

首先，这个问题，第一个让我联想到的是Java的CurrentSkipListMap，当然可以有CurrentTreeMap，但是Java8还没有。如果有，那有什么区别？其实本质都是数据结构的问题，skiplist与红黑树都支持范围的查询，而是时间复杂度也一样的。唯一的区别就是在频繁的插入或删除情况下，为了有更好的查找性能，树需要经常的旋转和调整，而调整的过程中，需要锁住很多的节点，这也是这个skiplist数据结构的特点，并发好。

引用一个回答:[How does lucene index documents?](https://stackoverflow.com/a/43203339)

> For those curious why Lucene uses Skip-Lists, while most databases use (B+)- and/or (B)-Trees, take a look at the right [SO answer](https://stackoverflow.com/questions/256511/skip-list-vs-binary-tree/28270537#28270537) regarding this question (Skip-Lists vs. B-Trees). That answer gives a pretty good, deep explanation - essentially, not so much make concurrent updates of the index "more amenable" (because you can decide to not re-balance a B-Tree immediately, thereby gaining about the same concurrent performance as a Skip-List), but rather, Skip-Lists save you from having to work on the (delayed or not) balancing operation (ultimately) needed by B-Trees (In fact, as the answer shows/references, there is probably very little performance difference between B-Trees and [multi-level] Skip-Lists, if either are "done right.")

分析下上面的答主关键点：

- 如果正确使用，B-Trees and [multi-level] Skip-Lists，性能差距是very little的
- 使用Skip-Lists的一个重要的好处是，从开发的角度上看，不用考虑什么时机去re-balance一个B-Tree(优化性能的一个关键)，从使用的角度上看，如果需要上层触发re-balance，那么使用者也是方便了很多。

如果说RBTree在更新的时候是需要锁住很多的节点的，那Skip-Lists呢？锁住的很少只需要相邻的节点即可。

另外，在[Skip List vs. Binary Tree](https://stackoverflow.com/a/28270537)中一个高票回答，也说明了，基于tree的实现在并发上能够达到Skip List的性能，但是，非常容易screw-up：

> Gramoli's conclusion is that's much easier to screw-up a CAS-based concurrent tree implementation than it is to screw up a similar skip list.

而且回答中还带了论文的数据，不过没打开。

总结，why Lucene use Skip List not B-Tree ? 在达到同样性能的条件下，使用Skip List更容易实现。

以上，是目前查到资料得到的答案，后续待补。



## 参考 ##

[Lucene学习总结之三：Lucene的索引文件格式 (1)](http://forfuture1978.iteye.com/blog/546824)  
[Lucene's RAM usage for searching](http://blog.mikemccandless.com/2010/07/lucenes-ram-usage-for-searching.html)  
[Apache Lucene - Index File Formats](http://lucene.apache.org/core/2_9_4/fileformats.html)  