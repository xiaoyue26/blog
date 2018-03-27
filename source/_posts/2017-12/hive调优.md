---
title: hive sql调优
date: 2017-12-25 16:45:15
tags: hive
categories:
- hadoop
- hive
---

这里记录一下hive任务调优的三(n)板斧.

# map join
对于存在join的sql,首先最简单的就是开启map join:
```sql
set hive.auto.convert.join = true ; -- 开启自动转化成mapjoin
set hive.mapjoin.smalltable.filesize = 2500000 ; -- 设置广播小表size
```
sql中足够小的表应该放在join操作左边. 由于小表数据会被广播到各个节点,消除了shuffle运算,提高了运算效率.
前提当然是存在足够小的表. 实际业务中一般是各种维度表.

# 排序消除
> 注: 是否加速取决于数据集.

排序属于非常耗时的操作(`O(nlogn)`),所以对于order by,sort by语句,可以从语义上寻找突破口. 例如对于每天最后一次的用户行为,原来的可能是这样写的:
```sql
SELECT *
(select userid
      ,url
      ,row_number(partition by userid order by bigint(timestamp)DESC) as rank
FROM xxxx
)AS a 
where rank=1
```
可以改为先求最大时间戳,再进行`join`(`map join`):
```sql
select * from
(select userid,max(bigint(timestamp)) from xxx group by userid) as a
join xxx on b 
  on a.timestamp=b.timestamp
```
这样更改后虽然消除了排序操作,但是引入了shuffle操作(join)(并且对于hive要laod两遍数据),因此是否加速取决于具体的数据集. 对于任务卡死(或者很慢)在reduce阶段的hive任务,可以尝试进行排序消除. 
实际经验来看,如果数据量大到导致外排,需要消除`order by`.

# distinct消除(两阶段group by)
回字有4种写法,而distinct一般有2种.

## 1. 多列或1列去重
```sql
select distinct a,b,udf(c1) as c2 from xxx
```
由于hive是通过`group by`实现distinct,上述sql其实等效于:
```sql
select a,b,udf(c1) from xxx group by a,b,udf(c1)
```
可以通过explain查看两者的执行计划是完全一致的.
如果能确定udf是单射变换,也就是c1到c2是一对一,而没有多对一,可以等效改写为:
```sql
select a,b,udf(c1) from xxx group by a,b,c1
```
总之,对于这个场景下的distinct使用,如果没有udf,可以不进行消除.

## 2. 聚合函数中使用(如uv计算)
```sql
select dt,count(distinct userid) as uv 
from xxx
group by dt
```
这种聚合函数中使用distinct属于比较常见的业务查询需求,hive执行时会把所有数据灌到一个reducer中,毫无并行度.
可以使用两阶段group by进行优化,写法:
```sql
select dt,count(1)
FROM
(select distinct dt,userid from xxx) as t 
group by dt
```
这样去重操作在第一个阶段分担到了多个reducer上,速度提升很多.

实际优化的时候,主要有三种情况阻碍,无法直接改写:
### 1. 同一列不同条件的count distinct
```sql
select dt
,count(distinct userid) as seven_uv
,count(distinct if(c1>xxx,userid,NULL)) as new_uv
,count(distinct if(c2>xxx,userid,NULL)) as query_uv
```
可以通过增加标记列转化:
```sql
select dt
,count(userid) as seven_uv
,count(if(is_new=1,userid,NULL)) as new_uv
,count(if(is_query=1,userid,NULL)) as query_uidnum -- query_uv
from 
(...
,max(if(c1>xxx,1,0)) is_new
,max(if(c2>xxx,1,0)) as is_query
...) as tt
```

### 2. 多维聚合(group by with cube)
可以通过一行变多行,手动维护grouping sets的组合:
```sql
lateral view explode(array('全部',platform)) tt1 as platform_t
lateral view explode(array('全部',version)) tt2 as version_t
lateral view explode(array('全部',vendor)) tt3 as vendor_t
lateral view explode(array('全部',phase)) tt4 as phase_t
GROUP BY platform_t,version_t,vendor_t,phase_t
     ,userid
```

## 不同列聚合.
例如:
```sql
count(distinct userid)
count(distinct deviceid)
```
这种如果确实出现了reduce卡死,可以进行分拆成两个查询分别计算(load两遍数据),最后join到一起. 代码会比较长. 



