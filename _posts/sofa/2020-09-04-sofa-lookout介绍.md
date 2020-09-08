---
layout: post
title: sofa lookout介绍
category: sofa
tags: sofa
---
## 介绍
首先这个名字lookout，动词是眺望的意思，名字就是能够眺望的地方，"a place from which to keep watch or view landscape."，翻译过来更像是看板的意思，作为一个监控系统，这意思体现了系统的高层次的监控一面。

- 作为一个监控系统，后端是默认通过ES来存储的，但ES好像并不善于对数据进行聚合、分析。难道分析聚合的场景不多？更多的只是把数据拿出来进行展示？
- 与mi公司的日志分析类似，默认的策略是存储近7天

## API使用
- 与falcon类似，数据的属性主要是通过tag的方式来区分，而且查询的时候会强制制定tag，这也间接说明监控项目不会很多（建议上限5000），每个监控下会有很多数据，需要通过tag来区分
- 有一些默认的tag，如优先级tag（决定了采用的间隔），可以不进行显示设置有默认值。感觉这名字与功能之间有些差距，为什么不叫采样间隔tag？
- 与falcon一样，有一些默认的监控项如机器的一些指标、jvm的状态


## Metric类型
- 这些类型是监控系统中通用的
- Counter（方法调用次数）、Timer（方法耗时）
- DistributionSummary，提供了一些简单的统计功能，如sum、平均、最大，更重要的是有分布直方图
- Gauge：“Different to other meters, Gauges should only report data when observed. Gauges can be useful when monitoring stats of cache, collections, etc.:”
- MixinMetric，一个指标的多个维度信息如线程池监控：有总线程、活跃、等待队列大小等
- Top工具，一些场景下，如列出Top 5/10等的度量信息，如top慢的sql，可以使用 TopUtil（它可以只记录前几名的数据，淘汰掉其他数据）

## 报警
没找到Lookout报警的相关功能，falcon中支持的策略有：

>all(#3): 最新的3个点都满足阈值条件则报警  
max(#3): 对于最新的3个点，其最大值满足阈值条件则报警  
min(#3): 对于最新的3个点，其最小值满足阈值条件则报警  
sum(#3): 对于最新的3个点，其和满足阈值条件则报警  
avg(#3): 对于最新的3个点，其平均值满足阈值条件则报警  
diff(#3): 拿最新push上来的点（被减数），与历史最新的3个点（3个减数）相减，得到3个差，只要有一个差满足阈值条件则报警  
pdiff(#3): 拿最新push上来的点，与历史最新的3个点相减，得到3个差，再将3个差值分别除以减数，得到3个商值，只要有一个商值满足阈值则报警  
exists(#2/3): 最新的3个点中有2个满足条件则报警，不同于open-falcon，open-falcon使用 lookup(#2,3)  


## 参考
[SOFA-prc](https://www.sofastack.tech/projects/sofa-rpc/getting-started-with-rpc/)






