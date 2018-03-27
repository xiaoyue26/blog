---
title: 高性能Mysql笔记-第六章(3)
date: 2018-03-12 20:31:17
tags: mysql
categories:
- mysql

---

- 前情提要:
前面讲了一些场景下针对性的优化。

## 6.7.5 优化Limit分页
- 场景
偏移量非常大，数据量非常的时候。

例如Limit 10000,20.
- 工作原理：
Mysql查询10020条记录，然后返回最后20条。前面1W条都被丢弃。

- 解决方案：
1. 限制分页的数量；(不让查这么大偏移量)
2. 使用覆盖索引优化性能。(延迟关联)
3. 记录上次分页的id。(利用索引)

- 方案2（延迟关联）
其中方案2，使用覆盖索引优化的原理：
1. 使用覆盖索引，返回目标行的id；
2. 再使用关联查询，返回目标行的所有列。

方案2在偏移量很大的时候，性能优化明显。

- 方案3（记录上次分页id）
这个方案在多次顺序分页的时候性能很好。
比如上次分页查询到了id为10240的地方，记录这个值（可以在应用层），然后下次的查询就可以是:
```sql
select * from t
where id>10240
limit 20
```

## 6.7.6 SQL_CALC_FOUND_ROWS相关
使用limit的时候，如果加上hint:
```sql
SQL_CALC_FOUND_ROWS
```
就可以获得limit之前结果集的行数。

- 工作原理：
Mysql扫描整个结果集，获得行数。
如果不用这个hint，则Mysql只扫描limit的行数返回即可，因此这个hint的代价可能非常大。

使用这个hint的可能原因及相关优化：
1. 想知道是否应该显示“下一页”的按钮；
解决方案：如果原来是limit 20,则改成limit 21即可，这样就知道需不需要“下一页”了。
2. 应用层进行缓存；
3. 使用Explain结果的行数估算。

## 6.7.7 优化UNION查询
- UNION
去重后合并
- UNION ALL
不去重，直接合并

- Mysql执行UNION的原理：
创建临时表，UNION的过程逐步填充行。

优化方法：
将WHERE,limit,ORDER BY子句置入内部UNION的子句，以提前过滤。

## 6.7.9 用户自定义变量
- 示例:
```sql
-- 定义：
set @one :=1;
set @min_actor := (select min(id) from actor); -- 赋值运算优先级很低，所以右边一般都要括号。
set @last_week := current_date-interval 1 week;

-- 使用：
select * from t
where col <= @last_week;
```

**缺点：**
1. 使用自定义变量后，无法使用查询缓存；
2. 表名，列名，Limit中无法使用；
3. 仅在当前连接中有用；（如果持久化连接、连接池，则会发生数据竞态，出BUG）
4. 不能定义变量的类型；
5. 优化器可能将变量优化掉，导致结果不可预期；
6. 赋值的顺序和时机不固定，导致结果不可预期；
7. 使用未定义的变量不会产生任何语法错误。（编译期无法判断错误）出错几率很大。

- 优点：

### 优化排名语句
实现行号功能:
```sql
set @rownum:=0;
select id,@rownum := @rownum+1 as rownum
from actor limit 3;

-- 结果：
id： id1 id2 id3
rownum: 1 2 3
```

### 查询并且更新数据
原查询：（2条语句）
```sql
update t set lastUpdated=now() 
where id=1;
select lastUpdated from t where id=1;
```
改写后：(2条语句)
```sql
update t set lastUpdated=now() 
where id=1 and @now:= now();

select @now; -- 无需访问数据表
```

### 统计更新和插入的数量
- 示例：
```sql
insert into t (c1,c2) values
(4,4)
,(2,1)
,(3,1)
on duplicate key update
c1=values(c1)
+( 0* (@x:= @x+1));

select @x;
```
第一句的返回值能知道更改了n1行；
第二句通过`@x`统计到更新了n2行；
因此：
更新的行数=n2;
插入的行数=n1-n2。

# 6.8 案例学习
## 6.8.1 使用MYSQL构建一个队列表
- tips:
无论什么时候，避免`Select .. For Update`这种用法。
原因：会造成事务阻塞。总是有更好的方法替代这个用法。
此外，对于索引列，它产生排他行锁；
对于非索引列，它产生排他表锁。

相关函数：
```sql
select connection_id();
-- 返回当前连接的id
```

案例：
表unsent_emails:
- id
- status (未发送，已发送，正在发送)
- owner （正在处理的连接id，默认是0）
- ts

错误示例：
```sql
Begin;
select id from unsent emails
where owner=0 and status='未发送'
limit 10 
For Update;
-- 结果是1,2,3
update unset_emails
set status='正在发送'
,owner=Connection_id()
where id in (1,2,3);
Commit;
```
select和update之间的间隙时间内，所有相同的查询会阻塞。（所有线程全都在竞争1个排他锁）

改进后：
```sql
set autocommit=1;
commit;
update unset_emails
set status='正在发送'
,owner=connection_id()
where owner=0 and status='unsent'
limit 10;

set autocommit=0;
-- 找出自己抢注的部分，进行处理：
select id from unsent_emails
where owner=connect_id() 
and status='正在发送';
-- 处理....
```

改进后，不必使用`select ... for update`，避免了事务阻塞。
唯一需要注意的时，需要定期检查是否有线程挂了，需要把它原来抢注的任务重置为无人拥有。

此外，消息队列还是别用Mysql实现了，用MQ,redis之类的吧。XD

## 6.8.2 计算两点之间的距离
- Tip:
复杂地理信息计算建议使用PostgreSql.

案例：
某个点附近所有可以出租的房子。
（一定半径内的所有点）
（或社交网站中匹配附近的用户）

表Locations:
- id
- name
- lat: 经度
- lon：纬度


A点和B点之间的距离计算公式`CAL`:
```sql
Acos(
    cos(latA)*cos(latB)*cos(lonA-lonB)
  +sin(latA)*sin(latB)
)* 地球的半径
```

**原查询：**
```sql
select * from locations
where 地球的半径* CAL <= 100;
```
上述查询无法使用索引，且非常消耗CPU。

优化方法：降低精度；修改上述的椭圆计算公式，近似成立方体公式。

**- 优化后：**
```sql
select * from locations
where lat between 38.03-degrees(0.0253) and 38.03 + degree(0.0253)
and lon between -78.48 - degrees(0.0253) and -78.48 + degrees(0.0253);
```
上述查询可以利用索引(lat)或(lon)，但由于都是范围查询，无法利用联合索引。

为了进一步优化范围查询，可以使用枚举优化（使用IN列表）。
由于lat和lon列其实不是离散值，可以手动创建离散列：
新增两列存储坐标的近似值Floor()，然后在查询中使用IN将所有点的整数值放在列表中：
- Lat_floor: floor(lat)
- Lon_floor: floor(lon)
- Key(Lon_floor,Lat_floor)

**- 进一步优化：**
```sql
select * from locations
where lat between 38.03-degrees(0.0253) and 38.03 + degree(0.0253)
and lon between -78.48 - degrees(0.0253) and -78.48 + degrees(0.0253)
-- 增加对于索引的利用：
and lat_floor in (36,37,38,39,40)
and lon_floor in (-80,-79,-78,-77);
```
上述优化可以利用索引(lat_floor,lon_floor)，因此比前面更快。

- 精度恢复：
上述优化只是滤掉不适合的点，可以进一步使用原来的公式进行过滤。由于此时剩下的点不多，因此即使原来的公式很复杂，性能也能够接受。

## 6.8.3 用户自定义函数(UDF)
书里说的比较含糊，总之是写一个C/C++程序作为后台程序，接受某种网络通信协议。
(TODO,进一步搜索别人的实际案例。)

# 6.9 总结
查询优化的思路：
1. 检查执行的时间消耗；
2. 检查Schema设计；
3. 检查索引设计；(三星系统)
4. 检查Explain计划（是否利用到索引，范围索引转化为枚举）
5. 其他的各种优化：
(1) limit优化；
(2) 覆盖索引+延迟关联；
(3) 复杂查询分解；
(4) 执行计划搜索提前终止；
(5) 排序优化（1次传输，2次传输），Order By的列提取到第一张表；
(6) UNION优化：手动下推谓词；
(7) Count优化：反向过滤， 利用MyISam的统计信息；
(8) Delayed优化：日志系统异步写入；
(9) 排他锁优化：避免使用`Select ... for update`，尽早释放锁（先抢注，释放锁，后处理）。