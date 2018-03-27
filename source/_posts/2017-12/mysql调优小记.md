---
title: mysql调优小记
date: 2017-12-07 18:49:56
tags: mysql
categories:
- mysql
---

- 摘要
> mysql表出现慢查询,单表数据量500W~700W条. 每天2~3W条.
措施: 
> 1. 优化查询语句; 
> 2. 更改存储引擎.
> 3. 优化索引.

### 详情
同事去塞班岛玩前留了个大坑,导致网站报表卡死刷不开.

### STEP 1
进入后台:
```sql
show processlist
```
发现Query很多,`show full processlist`找出查询语句,发现多表join的时候没有利用到索引.
查询语句模式如下:
```sql
explain
select *
from  a
LEFT join b
    on a.dt=b.dt
    and a.platform=b.platform
    and a.version=b.version
    AND a.vendor=b.vendor
where a.dt>='2017-12-01' and a.dt<='2017-12-07'
and a.platform='xxx' ... 
```
mysql没有通过语义上的相等,把加给a表的条件传递给b表,导致多表join的时候没有利用到索. 此时`explain`后得到的a表`type`是`range`而b表是`ALL`. 后续还有几个join也都是类似情况,因此把给它们分别都加上子查询:
```sql
explain
select ...
from (SELECT * FROM aa
          WHERE dt>='2017-12-01' and dt<='2017-12-07'
          AND phase='xx' AND grade='xx'
          AND vendor='xx'
          ) AS a
    left join
          (SELECT * FROM bb
          WHERE dt>='2017-12-01' and dt<='2017-12-07'
          AND phase='xx' AND grade=''
          AND vendor='xx'
          )AS b
    on a.dt=b.dt
    and a.platform=b.platform
    and a.version=b.version
    AND a.vendor=b.vendor
...
```

- `explain`查询计划中的`type`:
```sql
const(system): 根据PRI或Unique key,只取出确定的(一行)数据,常量优化. 

eq_ref: JOIN条件包括了所有索引,并且索引是Unique key. 
ref: JOIN条件包括部分索引或不是Unique key.
ref_or_null: WHERE col=exp or col is null

index_merge: Where id=xx or userid=xxx. 索引合并优化.

unique_subquery: where col in( select id from xxx). 这里id唯一.
index_subquery: 同上,只是id不唯一.

range: 索引在某个范围内.
all: 扫全表
```

- TODO:
研究除了子查询以外的方式使用索引.

此外, 索引对于数据类型敏感, 查询中存在字符串和date类型相等的时候, 无法利用索引,
需要将date类型转成字符串.
```sql
dt = date_format(date_sub(current_date, interval 1 day), '%Y-%m-%d')
```

### STEP 2

检查后台,发现许多连接状态都是:
```waiting for table level lock```.
进一步发现这几张表的存储引擎是`MYISAM`,而不是默认的`innodb`.
由于`myisam`引擎只有表级锁,不符合我们的使用要求.于是我把涉及到的几张表引擎都改为`innodb`.


### STEP 3

修改后,查询不会卡死(毕竟不扫全表了ORZ),降低到40s,但还是太慢了. 
进一步检查explain结果,发现`key_len`太长了. 
于是重新设计索引, 把筛选度高的放前面(利用最左前缀), 并且根据具体业务\语义,尽量缩短索引字段的长度,
实在无法缩短的则取其中一部分. 最后查询缩短到0.04sec,几个报表都是秒出.  


