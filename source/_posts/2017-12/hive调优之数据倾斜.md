---
title: hive调优之数据倾斜
date: 2017-12-26 19:35:00
tags:
---

上一篇中记录了hive调优的一些常规手段. 但对于某些数据集, 常规手段是无能为力的, 例如数据倾斜时.

对于hive而言,数据倾斜就是某个reducer跑得特别慢,这一点可以从日志中reducer开在99%或某个值很久看出,也可以从web ui中查看:
```
运行后:
http://xxx:19888/jobhistory/tasks/job_1472710912354_3070682/r
运行前:
http://xxx:8088/proxy/application_1472710912354_3070684/mapreduce/tasks/job_1472710912354_3070684/r
```
如果具体看日志的话,会发现大部分时间在进行外排. 
对于这种任务最重要的是消除外排,有如下几种优化手段:

## 1. 加内存
最简单粗暴就是给reduce加内存了. 让它别外排:
```
set mapreduce.reduce.memory.mb=10240;
```
类似的,如果mapper内存不够,可以减小每个mapper处理的数据量,增大mapper的数量:
```
set mapreduce.input.fileinputformat.split.maxsize=64000000;
```

## 2. 倾斜key单独处理
第二种手段也比较简单, 就是把出现倾斜的key找出来,假如很少的话,可以把它们摘出来,单独处理(或遗弃). 开启hive自动消除数据倾斜:(效果有效)
```
set hive.optimize.skewjoin = true;
set hive.skewjoin.key=1000;
set hive.groupby.skewindata=true;
set hive.groupby.mapaggr.checkinterval=100;
```

## 3. 局部聚合(1): 相同value聚合
(没有什么优化是增加一个阶段不能解决的.如果有,就再加一个阶段)
为了减少最后汇聚到reducer上的数据量,可以在之前增加一个阶段,对某个key的数据进行局部聚合. 

以某次需求为例,需要求各个省市区维度下的丢包率\延迟的50,90,99分位数.数据量每天200G. 分位数计算极其耗时, 尤其是计算周统计数据时, 数据量达到TB级. 

在使用了前一篇优化笔记手段以及上述手段后,依然耗时4小时.原来查询最耗时的部分如下:
```
select  es
       ,province
       ,city
       ,ipOprator
       ,percentile(xxx,0.5,0.9,0.99)
FROM ttt
GROUP BY es,province,ipOprator with cube
```
查看hive的percentile源码实现,其对于同一个key的处理逻辑大致这么几步:
```
(依次输入每个value)
1. O(n)
把所有value放进一个Map<value,LongWriteable>里,相同的value则增加map中的计数值;
2. O(nlogn)
在reduce中把map中的所有entry放入一个List中,然后对List根据value值进行全排序(`Collections.sort(entriesList, new MyComparator());`);
3. O(n)
从头开始扫一遍上一步的List, 根据计数器的值总和,分位数,定位到对应的分位数,返回.
```
将其重写为可以进行局部聚合, 从而略去第一步:
```
select  es
       ,province
       ,city
       ,ipOprator
      ,percent_new(c1,num,0.5,0.9,0.99)
FROM (select es,province,ipOprator,c1,count(1) as num
    FROM xxx
    GROUP BY es,province,ipOprator,c1
) as t 
GROUP BY es,province,ipOprator with cube
```
优化后,时间缩短到30分钟.

- TODO:
优化第二步中的全排序. 


## 4. 局部聚合2: 相同key聚合
由于上一案例中的聚合函数是分位数计算,聚合的粒度只能达到相同value聚合,对于其他聚合函数,如最大值,最小值等,如果语义上能对相同key先聚合,问题的规模就可以进一步缩小. 方法是先把相同key的数据分拆成不同的key,加上前缀或后缀 如:
```
key -> key_1
key -> key_2
...
key -> key_10
```
分拆的数量等于并行度,取决于原有的数据集, 然后先进行一阶段聚合,最后去掉前缀后缀,再进行一次聚合得到最后的结果. 
这种方法的关键就是要求同一个key的聚合计算可以分拆.