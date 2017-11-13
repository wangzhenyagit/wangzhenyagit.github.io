---
layout: post
title: MyISAM与InnoDB引擎
category: 存储系统
tags: mysql
keywords: mysql
---

具体分析待补充，先上几个结论：
- MyISAM与InnoDB都是B+树
- MyISAM叶子节点存储的是数据指针，而InnoDB叶子节点直接存储了数据
- 因为局部性原理与磁盘预读机制，node种没有data的B+树索引的效率更高
- InnoDB的主键最好是单调的，这样会顺序的写磁盘，不用调整树。否则插入时调整树会造成大量磁盘IO与碎片。


[MySQL和Lucene索引对比分析](http://www.cnblogs.com/luxiaoxun/p/5452502.html)
[MySQL索引背后的数据结构及算法原理](http://blog.codinglabs.org/articles/theory-of-mysql-index.html)