---
layout: post
title: 《Elasticsearch权威指南》读书笔记
category: 读书笔记
tags: ES
---

没买纸板的，直接读的网上的中文版本[Elasticsearch权威指南](https://www.elastic.co/guide/cn/elasticsearch/guide/current/index.html)，翻译的不好理解的有障碍可以参考下英文原版[Elasticsearch: The Definitive Guide](https://www.elastic.co/guide/en/elasticsearch/guide/current/index.html)。

很早以前有初步了解，直接带着问题找下答案。


## 过滤与查询基本概念 ##
这两个基本概念是写DSL语句中最关键的概念，可以不了解分片机制，集群如何运行，但这两个概念是学习ES的最基础的了。

直接引用原文中的[Queries and Filters](https://www.elastic.co/guide/en/elasticsearch/guide/current/_queries_and_filters.html)的部分。

> The DSL used by Elasticsearch has a single set of components called queries, which can be mixed and matched in endless combinations. This single set of components can be used in two contexts: filtering context and query context.
> 
> When used in filtering context, the query is said to be a "non-scoring" or "filtering" query. That is, the query simply asks the question: "Does this document match?". The answer is always a simple, binary yes|no.
> 
> When used in a querying context, the query becomes a "scoring" query. Similar to its non-scoring sibling, this determines if a document matches and how well the document matches.
> 
> A scoring query calculates how relevant each document is to the query, and assigns it a relevance _score, which is later used to sort matching documents by relevance. This concept of relevance is well suited to full-text search, where there is seldom a completely “correct” answer.

ES中的两种查询context: filtering context and query context.不难理解，一种判断是否匹配的filtering query，结果是确定的是或者否，而另外一种是需要计算分值的scoring query。

当然，最重要的是性能问题，也直接引用原文：

> Filtering queries are simple checks for set inclusion/exclusion, which make them very fast to compute. There are various optimizations that can be leveraged when at least one of your filtering query is "sparse" (few matching documents), and frequently used non-scoring queries can be cached in memory for faster access.
> 
> In contrast, scoring queries have to not only find matching documents, but also calculate how relevant each document is, which typically makes them heavier than their non-scoring counterparts. Also, query results are not cacheable.
> 
> Thanks to the inverted index, a simple scoring query that matches just a few documents may perform as well or better than a filter that spans millions of documents. In general, however, a filter will outperform a scoring query. And it will do so consistently.
> 
> The goal of filtering is to reduce the number of documents that have to be examined by the scoring queries.

只要有两点，一是由于不需要计算分值，filtering query相对于scoring query很轻的，而且由于结果大概率上是可以复用的，所以可以缓存。而scoring query是不缓存的，原因可能是因为复用的概率太小，只要查询词稍加变化，以前的缓存就没用了，而ES是java的，垃圾回收还是要考虑的。如果不缓存可能在排序的过程中，那些不需要的结果内存就释放了，而且对下次查询影响较小（不用释放缓存内存）。个人猜测。

## field data机制 ##
## ES的内存 ##
## 车牌+属性检索，如何优化 ##
## 在大数据下，与关系数据库比，精确查询，哪个更优 ##