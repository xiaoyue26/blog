---
title: 高性能Mysql笔记-第六章(2)
date: 2018-03-11 17:09:07
tags: mysql
categories:
- mysql

---


- 前情提要：
SQL
-hash(SQL)**查询缓存**
->SQL
-**语法解析器**-
->解析树
-**预处理**-
->解析树(合法,有权限)
-**优化器**-
->最优执行计划
->接下来就要查询执行引擎了：


## 6.4.4 查询执行引擎
- 工作原理
查询执行引擎根据执行计划，访问存储引擎的相关API。(Handler Api)
查询中每张表由一个Handler实例表示，抽象了底层存储引擎的具体实现。

## 6.4.5 返回结果给客户端
采用TCP传输结果给客户端，为了避免服务器存储太多的结果，每生成一部分结果，就可以向客户端逐步返回结果集。
相关参数：
```
SQL_BUFFER_RESULT
-- 结果集的缓冲区
```
此外，查询的结果如果可以缓存，也会缓存下来。


# 6.5 优化器的局限性

## 6.5.1 关联子查询
- 案例
```
Select * from film
where id in (
    select film_id 
    from film_actor
    where actor_id=1
);
```

优化器改写后：
```
select * from film
where exists(
    select * from film_actor
    where actor_id=1
    and film_id=film.id
);
```
通过explain命令，可以看到
执行流程是：
1. 对film进行全表扫描，得到所有id；
2. 使用id逐个排查film_actor表。

- 缺点
对于film表很大的时候，性能很差。

- 解决方案1
```
select film.*
from film
join film_actor
on id=film_id
where actor_id=1;
```
这个查询优化后，会使用较小的表作为驱动表，因此可以回避上述缺点。


- 解决方案2
```
select * from film
where id in
( select group_concat(film_id)
from film_actor
where actor_id=1
);
```
这个查询优化后，会先对film_actor进行处理，生成IN列表后，会先排序，然后二分查找。仅当film很大，而film_actor表很小的时候效率比原来的方案高。（换言之，不一定更高）

## 6.5.2 UNION的限制
外层的limit20无法提前传入内层，因此可以自己手动在内层先Limit一下，减少数据量。
```
(select * from t1
limit 20
)
UNION ALL
(select * from t2
limit 20
)
limit 20
```

**以下是几个不支持的特性：**

## 6.5.5 Mysql无法并行执行
Mysql无法利用多核。

## 6.5.6 哈希关联
Mysql的关联都是嵌套关联，不支持哈希关联。
不过可以通过建立哈希索引来曲线实现哈希关联。
Memory存储的索引都是哈希索引。

## 6.5.7 松散索引扫描
例如当索引为(a,b)，而查询条件中只有b的时候，由于不是最左前缀索引，因此无法利用索引，Mysql只能进行全表扫描。

如果有松散索引扫描：
先扫描a列第一个值对应的b列的范围；
再扫描a列第二个值对应的b列的范围；
以此类推，就能少扫描很多数据行。
（类似一个对a列值进行枚举的过程）

可惜Mysql还不支持松散索引扫描。


## 6.5.8 最大值和最小值优化
- 案例
```
select min(id) from actor
where first_name='PENELOPE'
```

其中first_name列没有索引，而id是主键列。

因此Mysql的处理方法是：
1. 进行全表扫描获取符合where的数据行；
2. 将上述数据中的id取min.

- 优化方案
由于数据是按主键顺序排列的，因此全表扫描的时候，顺序IO，顺序扫描，遇到的主键也是递增的。因此扫描时遇到的第一个符合条件的数据行，它的主键值一定是最小的。

- 优化后的SQL
```
select id from actor
use index(primary)
where first_name='PENELOPE' limit 1;
```
- 优点：
快。

- 缺点：
可读性差。依赖于底层数据的物理组织。
没人能看懂这里是要取最小值。

## 6.5.9 同一张表同时查询和更新
Mysql无法运行的SQL：
```
update t AS out_t
set cnt= (
    select count(1) from t as in_t
    where in_t.type=out_t.type
);
```
上述SQL的语意是将表中相似行（type相同）的数量，记录到所有行的cnt列中。

- 改写后可以运行：
```
update t 
join 
    (select type,count(1) as cnt
    from t
    group by type
    ) AS tmp using (type)
set t.cnt=tmp.cnt
```
上述SQL使用生成表（临时表）的形式绕过了同时进行查询和更新的限制，查询的是临时表，更改的是原表。

**上述是Mysql不支持的特性。**

# 6.6 查询优化器的提示(hint)
这些提示随版本可能会变化。
目前可以使用的一些提示如下：

- **High_priority/Low_priority**
（作用不大，只对使用表锁的引擎有效）
多个语句同时访问某一个表时，哪些语句的优先级高。
原理：
等待某个表锁的语句组成等待队列。
这个提示可以控制等待队列中的排列顺序。


- **Delayed**
（有用）
对于`Insert`和`Replace`有效。
相应处理流程：
1. 立即回复客户端；
2. 将插入的行数据放入缓冲区；
3. 表空闲时将数据批量写入。

适用场景： 日志系统；
缺点：
1. 并不是所有存储引擎都支持；
2. 导致`Last_insert_id()`无法正常工作；
3. 不知道什么时候数据才真正落地。(有效性不好保证)

- **Straight_join**
手动指定表的关联顺序。

- **SQL_SMALL_RESULT**/**SQL_BIG_RESULT**
告诉优化器对于`Group by`或`Distinct`查询如何使用临时表及排序。

SQL_SMALL_RESULT：
结果集会很小，将结果集放在内存中的索引临时表，以避免排序操作。

SQL_BIG_RESULT：
结果集会很大，建议使用磁盘临时表进行排序。（外排）

- **SQL_BUFFER_RESULT**
使用这个hint:
将查询结果放入一个临时表中，然后尽可能快地释放表锁。

不使用这个hint:
查询结果全部发送给客户端后，才释放表锁。


- **SQL_Cache/Sql_no_cache**
是否缓存SQL结果。(详见第七章)

- **SQL_CALC_FOUND_ROWS**
(不该使用)
返回结果中加上结果集的大小。（Limit之前）


- **For update**/**Lock in share mode**
(尽可能避免使用)
只对实现了行锁的存储引擎有效。

- **Use index/ignore index/force index**
(有用)
指定优化器使用或不使用哪些索引。

- **optimizer_search_depth**
(有用)
优化器穷举执行计划的限度。
如果查询长时间处于`statistics`状态，可以考虑调低此参数。

- **optimizer_prune_level**
默认开启。
是否根据扫描的行数决定跳过某些执行计划。

上述hint即使有用，也不应滥用，不应假设自己比优化器的选择更优，可以实验验证。
应注意到随着版本升级，有些hint可能过时。（优化器变得更聪明）

# 6.7 优化特定类型的查询

## 6.7.1 count查询优化
count(1)等效于count(*)。

1. 利用Myisam特性优化
- MyIsam特性：
统计信息中记录数据行数量，因此全表count极快。

案例：
```
select count(1)
from t
where id>5;
```
其中id上有索引。

优化后：
```
select  (select count(1) from t ) 
     - count(1)
FROM t
where id<=5；
```
优化前扫描的为>5的所有行；(可能很多)
优化后扫描的为<=5的所有行（最多5行）。

应当注意到上述优化只对`MyISam`引擎有用。

2. 使用近似值
使用explain中预估的行数。
3. 缓存、汇总表

## 6.7.2 优化关联查询
1. 关联键上有索引；
2. Group by,order by只涉及一张表的列；

## 6.7.4 优化Group by和Distinct
Group by和Distinct会互相转化。

无法使用索引时，Groupby的两种策略：
1. 用临时表进行分组；(内存排序)
2. 用排序进行分组。(外排)

可以通过hint干预优化器的选择。

**关联查询+GroupBy**
- 案例
```
select first_name,last_name,count(1)
from film_actor
join actor using (actor_id)
group by first_name,last_name
```

优化后：
```
select first_name,last_name,count(1)
from film_actor
join actor using(actor_id)
group by actor_id
```
上述查询group by和select列不同.
- 相关参数
```
SQL_MODE=ONLY_FULL_GROUP_BY
这种取值时，这类查询会直接返回错误。
```
上述优化之所以能更优，是因为：
1. ID唯一决定了姓名，而且姓名唯一；
2. ID列分组效率高，而且所在的表小。

取消GROUP BY排序：
使用`ORDER BY NULL`，否则结果会按分组的字段进行排序。