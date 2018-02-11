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

## MyQA ##
### 在大数据下，与MySQL比，精确查询，多字段（10个），任意组合查询，哪个更优？ ###
从数据结构上分析，MySQL如果要对10个字段的任意组合进行快速的检索，那么需要建很多的索引，最左前缀匹配原则对索引中的字段确实是无法支持的，如索引字段组合a+b+c，虽然顶上了索引a、索引a+b、索引a+b+c三个索引，但是对于a+c、b、c、b+c是支持不好的。所以要对10个字段进行快速的组合，要不少的索引，具体多少个，对MySQL还不是很熟。

对于一些场景，有没有必要建立也是一个问题，如果要检索的数据是包含了原始数据很大的一部分，那么是不建议建立索引的。但是对于检索这种场景，一般需要扫描到的行数是不大的。

如果是比全文检索，那么ES目前应该是很有优势的。

### 车牌+属性检索，如何优化 ###
从业务上分析，车辆的特性比较多：

- 只有车牌号一个全文检索的字段，其他的字段例如车牌颜色、车的颜色、车的类型、车牌类型、过车时间等。

这样，可以利用ES的filter query，进行快速过滤。

- 过车记录的特性，绝大多数（90%）为私家车，绝大多数为黑色和白色。

当然，隐含的信息还很多，由于是私家车，大部分是蓝牌车，很少一部分是绿牌车。然后车的大小的类型中小型车很大多数。这样在检索的时候，需要根据输入的条件而组成不同的DSL语句。比如要检索“绿色车牌的白色车”，那么车牌颜色，应该是term中排名第一个的，但是要检索“蓝色车牌的绿色车”，那么车的颜色应该是term中的第一个了。

基于以上特点还能继续优化，可以推测，过车记录中“车身为黑色和白色的，车牌为蓝色的小轿车和SUV”可能占到总量的80%以上，姑且叫做过车大表，这样就可以根据特点来进行分表（ES中可以多索引、多类型），那么不仅可以节省很多字段的重复，例如这个过车大表，可以不用存储车牌颜色。甚至继续细分，如分为“白色蓝牌小汽车”、“黑色蓝牌小汽车”、“白色蓝牌SUV”等等，这样，基本上车身颜色、车牌颜色、车类型、这些字段能够省去80%的存储空间，即使设计很有，这些字段都是1个字节的byte，如果有100亿的数据总量，那么可以节省100亿*3（车牌颜色、车身颜色、车量类型）个字节数据，大概是3G的储存空间。但这个不是重要的，更重的是，索引的体积会大大减小，从而节省内存和磁盘空间，索引小了，效率自然会提高。而且，由于一些表中的字段少了，由于要求“10个字段任意组合检索”，索引的数目会大大的减少。减少的索引（可以认为是Mysql的正向索引，也可以是ES的倒排索引）的总体积（包括个数与单体减少），不是简单的80%这么多，可能是几倍的大小。

为表重要，单独一行，个人感觉上面这分表的方式，无论是ES还是Mysql都是效率提升非常明显的，缺点么，就是复杂。但是这“高频高价值”的功能，是值得的。


然后，还有个技巧，如果就是要检索“全年时间段，最近出现的以苏A开始的车，时间降序”，第一页内，用户可能只看10条记录，那么没有必要去检索一整年的数据，甚至只检索最近的10分钟的车就可以找到一页的数据了。但是是检索“全年时间段，最近出现的以苏A开始的黑色车牌的车，时间降序”，缩小的时间段就不好估计了。

- 车牌的数量是有限的，也就是说，即使你的库中有1000亿的过车记录，你的库中唯一的车牌也就是2亿（2017年底中国车牌保有量），而且一个城市的车辆超过300w的只有7个。

利用此特性，能够先把模糊查询转成精确查询，提高查询速度。例如检索一个模糊的车牌“苏A？1234”，可以先从车牌库中检索出精确车牌来，例如检索出为“苏AB1234”和“苏AC1234”，那么可以直接从过车库中利用term直接替换模糊的车牌字段，因为全文检索的过车是需要进行分值计算的，因为车牌库数目较少（一个城市的项目可能百万级），而过车库很容易百亿级别，对于ES来说，计算分值是很大的开销，如果从过车库检索，可能需要进行百亿次分值计算，即使有缓存一个车牌对应的分值，例如已经缓存了“苏AB1234”对应检索的分值，下次计算的时候可能会对比下，如果在遇到“苏AB1234”的车牌，直接从缓存中获取对应的分值就行了。当然，8成ES是不会有这种缓存机制的，因为绝大多数下，从缓存中“查找对比”甚至比计算分值更消耗资源，因为一般的检索对象可能是很长的比如是日志和文章，对比两个几K长度的字符串是否一致非常浪费，而且更关键的是，一般情况下，缓存的命中率是非常低的，相同的文章，一般会存放两遍。

但是，这种把模糊转成精确车牌再去查询，是有场景的。一种使用的场景如下：车牌库中只有车牌信息，没有其他任何信息，从车牌库中找出精确的车牌后，在使用这些精确的车牌从过车表中查询。这个时候，有个问题，模糊检索的条件从车牌库中检索出来的车牌，可能非常多，最坏的可能，如果只输入个“苏A”，可能有百万之多，那么，拿这100w个车牌去过车库中filter检索么？如果返回的数目有限，可以先拿1000个去检索，没有在去检索下1000下，然后问题就复杂了。但是，用户输入的检索条件中的时间段可能只有1个小时，1个小时，假设只有1w的过车记录，不如直接去这1w记录中直接计算下分值。

对于上面方案的问题，方案可以改进下。比如，还有另外一个过车时间库，记录下每天，每周，每月那些车有过过车记录，从车牌库中拿出结果后，与这个时间+车牌过车的库先取下交集，减少下检索车的精确车牌。但是还是有一系列的问题，比如如果按照每天来聚合过车车牌，那么每天的过车数目可能是一个城市几十万级别的，过滤的效果不明显。

那么这种利用车牌库转成精确查询，然后从过车库查找适用于什么场景呢？场景就是用户输入的条件比较具体，比如输入了“苏A?1234”,这样，精确的车牌最多有34个。而且用户检索的时间段越长越好，比如输入的时间是1年，而不是1个小时，这样，通过这种精确的车牌匹配，能非常快速过滤掉没用的数据，能够大大的减少ES的计算分值的计算量。

最后，这个技巧的场景用个例子总结下，适用于输入“苏A？1234”并且时间段为1年的检索，不适用于输入“苏A？”检索时间为1个小时的场景。

> 公安部交管局统计显示，２０１７年，全国汽车保有量达２．１７亿辆，与２０１６年相比，全年增加２３０４万辆，增长１１．８５％。从车辆类型看，载客汽车保有量达１．８５亿，其中以个人名义登记的小型和微型载客汽车（私家车）达１．７０亿辆，占载客汽车的９１．８９％；载货汽车保有量达２３４１万辆，新注册登记３１０万辆，为历史最高水平。从分布情况看，全国有５３个城市的汽车保有量超过百万辆，２４个城市超２００万辆，北京、成都、重庆、上海、苏州、深圳、郑州７个城市超３００万辆。

参考：[2017年底：我国机动车保有量３．１０亿辆 驾驶人３．８５亿人](http://www.xinhuanet.com/fortune/2018-01/15/c_1122262121.htm)

- 用户检索，返回的数据可能只看前几页

这里继续做业务上的假设：假设用户检索的是全部匹配的，且不需要计算分值的，按照时间降序的结果。啥意思？如果检索条件是“苏A 11”，给个时间段是过去一年，那么默认的ES的检索意思是，检索过去一年内，匹配关键此“苏A”或“11”的分值最大的top10结果，这就需要计算包含“苏A”或包含“11”或两者都包含的车牌，然后计算所有的车牌的相似度，然后排序取top10。想着就累。如果做了业务假设“假设用户检索的是全部匹配的，且不需要计算分值的，按照时间降序的结果。”，那么可以利用两个term进行快速的过滤车牌，不用计算分值，取得top10。

ES中有terminate\_after的功能，假设参数值设置成10，意思是找到10个匹配的后会立刻返回，当然也会根据分值从高到低，但是显然不是最高的分值，这个计算分值意义也不到。可以直接两个term+terminate_after，更加快速的返回结果，但是，不是“时间最近”的。

很像Mysql的limit功能，比如select * from table order by pass\_time limit 10,如果在pass_time中有降序的单列索引，那么就是根据索引从前面取出10条返回就可以了。

但是ES中没有类似Mysql中的B+树索引在单字段上，terminate_after这个是以什么顺序分析的呢？是倒排索引中的id的从小到大，还是是更复杂的策略呢？如果是id的从小到大，如果每次检索都是时间降序，那么可以让id是降序单调的，而不是升序单调的。**（待研究）**

如果没有类似Mysql的索引，那取出时间段内的完全匹配top10，可以分段进行，比如搜索“苏A？”，如果是在南京，那么估计过去1分钟内就会有10条结果，即使用户是检索一年的，可以后台只检索一分钟的数据，如果没有，变成检索2分钟的，ES会缓存，可以不用合并两次结果，直接在查询就好了。

### 测试调优 ###
在ES中数据达到10亿的时候，index的速度特别慢，只有大概250条每秒，iostat看了下是IO的问题，目前所有的分片只在一块盘上，理论上分析，如果分为5块盘，会有一定的速度提升，机械硬盘在并发IO下效率会较低，如果并发降低，效率会上升。这么多IO的问题主要是segment比较多，比如车牌会有20个segment文件，再加上10个左右的字段，有30个segment文件，这还只是一个分片，5个分片会有150个文件可能要更新。

然后，按照三个term的的方式查询，其实最后还是IO问题，而且竟然与时间跨度关系不大一个月（1亿数据）与10个月（10亿数据）的响应时间差不多，主要是读取segment文件，读上来索引文件，多个filter取交集非常快。

额，看来IO也是ES的非常重要的一个因素。