## 系统配置 ##

CentOS Linux release 7.4.1708 (Core)，128G内存，12*4T 7.2k机械磁盘。

ES版本5.6.8，heap使用32G，放在单块xfs磁盘上，无RAID，5分片，无副本。

### mapping ###

```
{
	"properties": {
		"land_id": {
			"type": "byte"
		},
		"plate_category": {
			"type": "byte"
		},
		"plate_color": {
			"type": "byte"
		},
		"vehicledim_id": {
			"type": "keyword"
		},
		"vehicle_category": {
			"type": "byte"
		},
		"vehicle_type": {
			"type": "byte"
		},
		"vehicle_color": {
			"type": "byte"
		},
		"event_type": {
			"type": "byte"
		},
		"vehicle_plate": {
			"type": "keyword"
		},
		"speed": {
			"type": "short"
		},
		"vehicle_length": {
			"type": "byte"
		},
		"bayonet_id": {
			"type": "keyword"
		},
		"pass_time": {
			"type": "date",
			"format": "yyyy-MM-dd'T'HH:mm:ss"
		}
	}
}
```

type的_id为event_id，使用snowflake算法生成event_id，41位时间戳，10位serverid，12位seq，起始时间2015年1月1日。

## 测试数据构建 ##

要求测试能够回归测试，能回归要求测试的检索数据不能随机，要提前生成好测试元数据。

### 元数据属性 ###
#### 卡口id ####
1000个不重复的id，id范围单调递增从“11000001802000000”到“11000001802000999”。

### 车信息表表 ###
100w个不重复的车辆信息，其中数据分布有如下规则：
车牌规则：
- 包括32个省市自治区的车牌
- 95%车牌为本省的车牌（以“苏”开始的车牌占总记录的95%）
- 如果为本省车牌90%为当地城市车牌（以“苏”开始的车牌中，其中“苏A”开始的占90%）
- 其他省份，一个省内的城市，车牌均匀分布

其他规则：
- 车牌类型，普通车牌，占所有总数的90%，其他车牌类型如农用车，摩托车等均匀分布
- 车牌种类，蓝色车牌，占所有总数的90%，其他车牌如黑色、绿色均匀分布
- 车辆类型，机动车占95%
- 车辆大小类型，小型车70%,中型车20%,大型车10%，小型车为2米和3米，概率50%；中型车为2米和3米，3米概率80%；大型车为4米和5米，概率50%；
- 车身颜色，白色车50%，黑色车30%，其他20%均匀分布

### 过车数据构建 ###

数据量为每秒100条过车记录，一天864w，一个月2.6亿，一年3.1亿数据。

过车信息与过车卡口信息从元数据表中随机抽取。时间跨度为一年，这样数据总量为365*3600*100=31亿。

构造数据过程中瓶颈为磁盘IO，CPU使用率2000%左右，使用15线程，每批1w条记录的速度构建数据，开始时候总速度有8w条每秒，随着数据的增加索引的速度下降，最后速度大概有4w+每秒，构造31亿数据总耗时大概17小时，平均速度5w条每秒。

## 预热方案 ##
按照如下步骤预热：
- 使用相同车牌号，使用所有的1000个卡口，每次10个卡口id作为过滤器进行bool查询
- 使用相同的车牌号，+每个参数，如车牌颜色、车身颜色等取值作为过滤器进行bool查询
- 随机车牌+随机个数个随机卡口id+随机个数个其他参数的随机合理范围内的值，进行1000次甚至更多的查询

## 测试结果记录统计 ##
- 统计响应时间分布的情况，50%的响应时间，60%的响应时间，70%的响应时间，80%的响应时间，90%的响应时间，95%的响应时间，99%的响应时间。
- 统计不同并发（同时查询的线程数）下，响应时间与吞吐量（暂不测试）

## 修改配置后测试&QA ##
- 目前是只使用一块磁盘，可以改成使用5块盘后测试
- 把模糊的车牌先转成精确的车牌，然后使用flitering查询，bool中should为3个词、2个词、1个词的，加上from与size，是否找到足够厚就返回？
- 32G内存外还会占用内存？
- 16G内存测试
- 10块盘测试
- 加上批量插入后测试

## 分析与结论 ##

### 检索数据 ###
检索要求，能够通过车牌+车辆类型+车牌类型+车型+车牌颜色+车身颜色+车速，组合条件进行检索，其中，车辆类型3种，车牌类型30种，车型4种，车牌颜色5种，车身颜色13种。只对车牌全文索引，最多三个关键词。

测试数据生成要求：

- 车牌类型有模糊的有精确的，关键词长度为7的为精确匹配。检索时候关键词为1个、2个、3个的比例为1:1:1。关键词长度为1、2、3、4、5、6、7的比例相等。
- 条件组合要全面，算上车牌有7个条件，组合条件较多，有C71+C72+...C77共种127种组合，要都覆盖到。比例相等。
- 车辆类型3种，车牌类型30种，车型4种，车牌颜色5种，车身颜色13种。检索请求中，各个参数值出现的比例相等。

## 测试结果详细附录 ##

### 对卡口id进行预热 ###
使用Java High Level REST Client5.6版本。

ES占用40G内存，操作系统缓存如下，应该大部分都是ES的文件缓存，前top10都是ES的文件。

              total        used        free      shared  buff/cache   available
Mem:           125G         99G        867M        526M         25G         24G
Swap:          4.0G        4.0G          4K

预热后跑用例：

随机车牌+n个随机卡口测试
BayonetNum : 1
Complete num:       1000
Time per search:    569 [ms](mean)
Time max:           1183 [ms]
Time min:           33 [ms]
Time sum:           569560 [ms]
Percentage of the requests served within a certain time (ms)
    5%    398
   10%    446
   20%    490
   30%    523
   40%    548
   50%    573
   66%    612
   75%    636
   80%    654
   90%    690
   95%    731
   98%    816
   99%    873
  100%    1183

BayonetNum : 2
Complete num:       1000
Time per search:    681 [ms](mean)
Time max:           1237 [ms]
Time min:           24 [ms]
Time sum:           681081 [ms]
Percentage of the requests served within a certain time (ms)
    5%    367
   10%    527
   20%    591
   30%    627
   40%    657
   50%    685
   66%    742
   75%    775
   80%    796
   90%    848
   95%    901
   98%    952
   99%    1016
  100%    1237

BayonetNum : 4
Complete num:       500
Time per search:    816 [ms](mean)
Time max:           2329 [ms]
Time min:           221 [ms]
Time sum:           408300 [ms]
Percentage of the requests served within a certain time (ms)
    5%    474
   10%    594
   20%    679
   30%    716
   40%    752
   50%    786
   66%    853
   75%    904
   80%    939
   90%    1067
   95%    1254
   98%    1458
   99%    1685
  100%    2329

BayonetNum : 8
Complete num:       500
Time per search:    1023 [ms](mean)
Time max:           2451 [ms]
Time min:           285 [ms]
Time sum:           511542 [ms]
Percentage of the requests served within a certain time (ms)
    5%    568
   10%    728
   20%    852
   30%    907
   40%    956
   50%    997
   66%    1080
   75%    1133
   80%    1171
   90%    1356
   95%    1583
   98%    1752
   99%    1836
  100%    2451

BayonetNum : 16
Complete num:       500
Time per search:    1048 [ms](mean)
Time max:           1910 [ms]
Time min:           445 [ms]
Time sum:           524024 [ms]
Percentage of the requests served within a certain time (ms)
    5%    685
   10%    786
   20%    947
   30%    996
   40%    1029
   50%    1060
   66%    1106
   75%    1139
   80%    1162
   90%    1251
   95%    1351
   98%    1500
   99%    1623
  100%    1910

BayonetNum : 32
Complete num:       500
Time per search:    1621 [ms](mean)
Time max:           2651 [ms]
Time min:           957 [ms]
Time sum:           810735 [ms]
Percentage of the requests served within a certain time (ms)
    5%    1215
   10%    1333
   20%    1451
   30%    1534
   40%    1586
   50%    1625
   66%    1688
   75%    1730
   80%    1767
   90%    1883
   95%    2031
   98%    2256
   99%    2340
  100%    2651

BayonetNum : 64
Begin test case, case num : 500
Complete num:       500
Time per search:    2397 [ms](mean)
Time max:           4129 [ms]
Time min:           1609 [ms]
Time sum:           1198915 [ms]
Percentage of the requests served within a certain time (ms)
    5%    1963
   10%    2069
   20%    2165
   30%    2231
   40%    2296
   50%    2343
   66%    2446
   75%    2498
   80%    2551
   90%    2715
   95%    3245
   98%    3621
   99%    3767
  100%    4129

BayonetNum : 128
Begin test case, case num : 500
Complete num:       500
Time per search:    3669 [ms](mean)
Time max:           8227 [ms]
Time min:           2647 [ms]
Time sum:           1834635 [ms]
Percentage of the requests served within a certain time (ms)
    5%    3077
   10%    3183
   20%    3291
   30%    3374
   40%    3443
   50%    3516
   66%    3660
   75%    3745
   80%    3810
   90%    4033
   95%    5518
   98%    6064
   99%    6376
  100%    8227

BayonetNum : 256
Begin test case, case num : 500
Complete num:       500
Time per search:    6022 [ms](mean)
Time max:           12899 [ms]
Time min:           4711 [ms]
Time sum:           3011242 [ms]
Percentage of the requests served within a certain time (ms)
    5%    5141
   10%    5235
   20%    5392
   30%    5522
   40%    5665
   50%    5787
   66%    5984
   75%    6143
   80%    6251
   90%    6793
   95%    8538
   98%    9675
   99%    10344
  100%    12899

BayonetNum : 512
Begin test case, case num : 500
Complete num:       500
Time per search:    10033 [ms](mean)
Time max:           19324 [ms]
Time min:           7732 [ms]
Time sum:           5016669 [ms]
Percentage of the requests served within a certain time (ms)
    5%    8382
   10%    8595
   20%    8946
   30%    9241
   40%    9454
   50%    9707
   66%    10194
   75%    10548
   80%    10822
   90%    11549
   95%    12389
   98%    15678
   99%    17880
  100%    19324

BayonetNum : 1000
Begin test case, case num : 500
Complete num:       500
Time per search:    9422 [ms](mean)
Time max:           22022 [ms]
Time min:           4281 [ms]
Time sum:           4711492 [ms]
Percentage of the requests served within a certain time (ms)
    5%    7284
   10%    7537
   20%    7886
   30%    8321
   40%    8635
   50%    8956
   66%    9820
   75%    10295
   80%    10680
   90%    11472
   95%    12201
   98%    16549
   99%    18359
  100%    22022

跑了一天的无查询，一直100条每秒的插入后，再次测试，与预热后的效果差很多：
BayonetNum : 1
Complete num:       500
Time per search:    1581 [ms](mean)
Time max:           4791 [ms]
Time min:           251 [ms]
Time sum:           790715 [ms]
Percentage of the requests served within a certain time (ms)
    5%    646
   10%    720
   20%    820
   30%    892
   40%    981
   50%    1085
   66%    1356
   75%    1648
   80%    2181
   90%    3828
   95%    4013
   98%    4179
   99%    4292
  100%    4791  

修改为使用5块磁盘，执行过程，磁盘使用率在20%一下，在64卡口的时候，cpu使用率380%，256卡口的时候，cpu使用率400+%，在1000卡口的时候，5块盘IO有50%。

BayonetNum : 1
Begin test case, case num : 500
Complete num:       500
Time per search:    75 [ms](mean)
Time max:           659 [ms]
Time min:           2 [ms]
Time sum:           37772 [ms]
Percentage of the requests served within a certain time (ms)
    5%    19
   10%    24
   20%    33
   30%    40
   40%    48
   50%    75
   66%    105
   75%    112
   80%    115
   90%    123
   95%    133
   98%    148
   99%    156
  100%    659

BayonetNum : 2
Begin test case, case num : 500
Complete num:       500
Time per search:    78 [ms](mean)
Time max:           192 [ms]
Time min:           19 [ms]
Time sum:           39242 [ms]
Percentage of the requests served within a certain time (ms)
    5%    32
   10%    37
   20%    44
   30%    50
   40%    56
   50%    64
   66%    102
   75%    114
   80%    121
   90%    133
   95%    141
   98%    154
   99%    162
  100%    192

BayonetNum : 4
Begin test case, case num : 500
Complete num:       500
Time per search:    88 [ms](mean)
Time max:           213 [ms]
Time min:           26 [ms]
Time sum:           44322 [ms]
Percentage of the requests served within a certain time (ms)
    5%    49
   10%    54
   20%    61
   30%    65
   40%    71
   50%    76
   66%    89
   75%    111
   80%    124
   90%    146
   95%    156
   98%    173
   99%    185
  100%    213

BayonetNum : 8
Begin test case, case num : 500
Complete num:       500
Time per search:    105 [ms](mean)
Time max:           218 [ms]
Time min:           55 [ms]
Time sum:           52951 [ms]
Percentage of the requests served within a certain time (ms)
    5%    69
   10%    74
   20%    81
   30%    86
   40%    91
   50%    96
   66%    107
   75%    119
   80%    130
   90%    162
   95%    173
   98%    188
   99%    196
  100%    218

BayonetNum : 16
Begin test case, case num : 500
Complete num:       500
Time per search:    133 [ms](mean)
Time max:           717 [ms]
Time min:           70 [ms]
Time sum:           66887 [ms]
Percentage of the requests served within a certain time (ms)
    5%    92
   10%    99
   20%    108
   30%    112
   40%    118
   50%    123
   66%    134
   75%    143
   80%    153
   90%    186
   95%    203
   98%    225
   99%    262
  100%    717

BayonetNum : 32
Begin test case, case num : 500
Complete num:       500
Time per search:    479 [ms](mean)
Time max:           792 [ms]
Time min:           364 [ms]
Time sum:           239747 [ms]
Percentage of the requests served within a certain time (ms)
    5%    416
   10%    429
   20%    442
   30%    454
   40%    465
   50%    473
   66%    487
   75%    502
   80%    511
   90%    536
   95%    574
   98%    612
   99%    654
  100%    792

BayonetNum : 64
Begin test case, case num : 500
Complete num:       500
Time per search:    805 [ms](mean)
Time max:           1470 [ms]
Time min:           643 [ms]
Time sum:           402639 [ms]
Percentage of the requests served within a certain time (ms)
    5%    718
   10%    731
   20%    755
   30%    771
   40%    785
   50%    796
   66%    816
   75%    828
   80%    838
   90%    888
   95%    919
   98%    978
   99%    1107
  100%    1470

BayonetNum : 128
Begin test case, case num : 500
Complete num:       500
Time per search:    1344 [ms](mean)
Time max:           1805 [ms]
Time min:           1127 [ms]
Time sum:           672113 [ms]
Percentage of the requests served within a certain time (ms)
    5%    1220
   10%    1245
   20%    1272
   30%    1296
   40%    1314
   50%    1334
   66%    1362
   75%    1383
   80%    1401
   90%    1457
   95%    1534
   98%    1632
   99%    1678
  100%    1805

BayonetNum : 256
Begin test case, case num : 500
Complete num:       500
Time per search:    2319 [ms](mean)
Time max:           2847 [ms]
Time min:           2049 [ms]
Time sum:           1159755 [ms]
Percentage of the requests served within a certain time (ms)
    5%    2197
   10%    2211
   20%    2239
   30%    2267
   40%    2289
   50%    2306
   66%    2338
   75%    2359
   80%    2374
   90%    2430
   95%    2534
   98%    2662
   99%    2720
  100%    2847

BayonetNum : 512
Begin test case, case num : 500
Complete num:       500
Time per search:    4077 [ms](mean)
Time max:           5229 [ms]
Time min:           3597 [ms]
Time sum:           2038633 [ms]
Percentage of the requests served within a certain time (ms)
    5%    3842
   10%    3899
   20%    3956
   30%    4001
   40%    4038
   50%    4068
   66%    4120
   75%    4151
   80%    4171
   90%    4251
   95%    4357
   98%    4559
   99%    4623
  100%    5229

BayonetNum : 1000
Begin test case, case num : 500
Complete num:       500
Time per search:    1486 [ms](mean)
Time max:           12433 [ms]
Time min:           924 [ms]
Time sum:           743041 [ms]
Percentage of the requests served within a certain time (ms)
    5%    1030
   10%    1088
   20%    1203
   30%    1347
   40%    1418
   50%    1465
   66%    1555
   75%    1626
   80%    1662
   90%    1744
   95%    1858
   98%    2068
   99%    2239
  100%    12433

https://discuss.elastic.co/t/what-are-field-evictions/1017
https://www.elastic.co/guide/en/elasticsearch/reference/master/circuit-breaker.html