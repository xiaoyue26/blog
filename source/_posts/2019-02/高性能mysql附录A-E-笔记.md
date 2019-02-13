---
title: 高性能mysql附录A-E-笔记
date: 2019-02-13 09:36:10
tags:
- mysql
categories:
- mysql

---


# 高性能mysql附录A-笔记-mysql分支
mysql的三个分支(变种):
1. Percona Server: 透明、性能、灵活，用XtraDB引擎代替innodb；
2. MariaDB: mysql原作者。面向客户、补丁插件扩展更多；
3. Drizzle：sql语法不兼容mysql, 修正bug，最开源。

官方mysql: 最接近于Percona Server。

# 高性能mysql附录B-笔记-服务器状态
系统变量: `show variables like '%xxx%'`;
只读状态: `show status like '%xxx%'`; 或`infomation_schema.global_status`表和`information_schama.session_status`表;
其他信息:`infomation_schema`库中

## 线程和连接统计
```
connections,max_used_connections,threads_connected
bytes_received,bytes_sent
slow_launch_threads,threads_cached,threads_created,threads_running
```

## 二进制日志状态
```
binlog_cache_use,binlog_cache_disk_use
```

## 命令计数器
`Com_*`变量统计每种类型的SQL或C API命令发起过的次数。
```
Com_select: select语句的数量;
Com_change_db: 更改默认数据库的次数(use xxx);

```

## 临时文件和表
隐式表和文件:
```sql
show global status like 'Created_tmp%'
```
显式临时表:
```sql
show global temporary tables;
```

## 查看select查询类型统计
```sql
show global status like 'Select%';
+------------------------+----------+
| Variable_name          | Value    |
+------------------------+----------+
| Select_full_join       | 677      |
| Select_full_range_join | 0        |
| Select_range           | 135124   |
| Select_range_check     | 0        |
| Select_scan            | 16623726 |
+------------------------+----------+
```
按预期次数从多到少/开销从少到多的顺序:
Select_range: 在第一个表上扫描一个索引区间的连接数目；
Select_scan: 扫描整个第一个表；
Select_full_range_join: 开销多于Select_scan；
Select_range_check: 开销非常高；
Select_full_join: 开销最高。

## 表锁
Table_locks_immediate: 立即授权的表锁次数；
Table_locks_waited:    需要等待的表锁次数。

## innodb状态
查看innodb相关的状态开销很大，会创建一个全局锁。
```sql
1. 通过show engine innodb status;
2. 通过information_schema表;
3. 通过show status,show variables。
```
因此不能频繁查看这些变量。

输出信息包括:
1. fsync()平均每秒调用次数；
2. 头部信息: 时间；
3. Semaphores: 操作系统等待数组，等待互斥量的innodb线程；(如果有等待，可以看出热点是什么)
```sql
waited at buf0buf.ic for 0 second: 等待缓冲区
waiters flag 0: 0个线程在等待；
waiting is ending: 等待结束。
```

`innodb_sync_spin_loops`变量:
空转多少次后停止spin，挂起进入真正等待。
```sql
show variables like '%spin%';
+------------------------+-------+
| Variable_name          | Value |
+------------------------+-------+
| innodb_spin_wait_delay | 6     |
| innodb_sync_spin_loops | 30    |
+------------------------+-------+

```

### 死锁
`SHOW ENGINE INNODB STATUS\G`的输出还有一部分是上两次死锁的情况。(包括进程id和sql、等待的是什么锁)

死锁类型：
1. 循环等待；
2. 等待关系图太深：
(1)检查超过100W个锁;
(2)重做超过200个事务。
错误信息："TOO DEEP OR LONG SEARCH IN THE LOCK TABLE WAITS-FOR GRAPH".

#### 减少死锁的TIPS
在事务里更新数据时，先按主键排序，这样扫描索引的顺序就都是一致的；


### 事务信息
`SHOW ENGINE INNODB STATUS\G`的输出里还有一部分是事务总结信息和当前活跃事务信息。
#### 事务总结信息
1. 当前事务ID;
2. 已经清理MVCC行的事务ID;(可以知道有多少老版本数据没被清理)
3. 历史记录的长度；
4. 锁结构的数目（可能包含多个行锁）。

#### 活跃事务信息
1. 进程id(与`show processlist`中id通用)
2. 内部查询号;
3. 连接信息
4. 查询的sql。

### File I/O
`SHOW ENGINE INNODB STATUS\G`的输出里还有一部分是IO辅助线程的状态和性能计数器的状态。
其中:
```sql
insert buffer thread: 插入缓冲合并到表空间；
log thread: 异步刷日志；
read thread: 预读操作；
write thread: 刷脏缓冲。
```

### redo log统计
`SHOW ENGINE INNODB STATUS\G`的输出里还有一部分是事务日志(redo log)的统计。
1. sequence number: 当前日志序号；
2. flushed up to  : 当前刷到哪里；  
3. last checkpoint: 上一个检测点的位置。
4. pending和done的日志操作数量。

### 缓冲池和内存
`SHOW ENGINE INNODB STATUS\G`的输出里还有一部分是BufferPool和内存统计。
包括信息：
1. 分配了多少字节；
2. pool总共多少页，其中free多少页；
3. database用了多少页，多少页已经修改；
4. 命中率等统计信息。
多个缓冲池的话后头还有各个buffer pool各自的统计信息。

### ROW OPERATIONS
`SHOW ENGINE INNODB STATUS\G`的输出里最后一部分是行操作统计。
```
--------------
ROW OPERATIONS
--------------
0 queries inside InnoDB, 0 queries in queue
15 read views open inside InnoDB
Main thread process no. 9004, id 139699662354176, state: sleeping
Number of rows inserted 1111442430, updated 2610575313, deleted 42594651, read 8977633484
98.47 inserts/s, 1108.97 updates/s, 0.00 deletes/s, 1671.23 reads/s
----------------------------
```
包括信息：
1. 多少线程在innodb内核内，多少在等待队列;
2. 累积增删改查;
3. 增删改查的瞬时速度。

## 主备相关查看命令
```
show master status \G
show binary logs;
show binlog events ... 
show relaylog events
show slave status
```

## Information_schema中的innodb信息表
## innodb_trx,innodb_locks
事务、拥有和等待锁的事务。

## performance_schema
略

# 高性能mysql附录C-笔记-大文件传输
大概是ssh,tar,gz,rsync命令的运用，略。
# 高性能mysql附录E-笔记-explain
explain语句：
1. `explain select xxx`: 近似的执行计划信息；
2.  近似的执行计划信息+等效sql:
```sql
explain extended select xxx;
show warnings; -- 这里得到的sql是从执行计划反向翻译过来的sql
```
3. `explain partitions select xxx`: 显示查询的分区。（仅对分区表有效）

mysql5.6前，explain的时候会执行子查询创建临时表，以便进行外层优化。

## explain各列的含义
## id
select语句顺序编号，对应在原始语句中的位置;（从外到内）

select的三种类型：
1. 简单子查询；
2. 复杂子查询（派生表）；
3. union查询。

其中union查询的id列为null，select_type列为`union result`。

## select_type列
取值含义：
```
simple:  简单查询;
primary: 复杂查询的最外层;
subquery: select列表中的子查询;(不在from子句中);
derived:  from子句中的子查询;
union:  union语句中第二个和随后的select;
union result: 从匿名临时表检索结果的select.
```

## table列
访问的表或别名。
或者< derivedN>,其中N是子查询的id。
union行中table列出现的< union2,3>其中2，3也是子查询的id。

## type列
访问类型：从差到好：

```sql
all: 扫全表
index: 按索引顺序扫全表(避免排序) -- extra列的using index表示覆盖索引
range: 有范围限制的索引扫描
ref:   索引查找，查找某个索引值；
eq_ref: 唯一索引查找，结果只有一行；
const,system: 可以优化成常量替换；
NULL: 可以优化到无需访问表和索引。
```

## possible_keys列
优化早期尝试的索引，可能无用。
## key列
决定采用的索引。
## key_len列
采用索引的可能最大长度。
## ref列
在索引查找值时使用的列或常量。
## rows列
估算大概要读取的行数。
## filtered列
`explain extended select xxx`时有的列。
符合条件行数的悲观估计。
## Extra列
包括的值：
```sql
using index: 使用覆盖索引；
using where: 从存储引擎返回后，需要在服务器再过滤一次;(可能暗示查询可以加更多索引)
using temporary: 查询结果排序时会使用临时表；
using filesort: 排序时会使用外部索引排序，而不是按索引顺序扫描；
range checked for each record: 没有理想的索引，对于连接后的每一行重新检查使用哪个索引(很慢)。

```

可以用`Percona Toolkit`的`pt-visual-explain`获得树形执行计划。

# 高性能mysql附录E-笔记-锁的调试
锁的类型：
1. 表锁；
2. 全局锁： `flush tables with read lock`或设置`read_only=1`
3. 命名锁：表锁的一种，重命名或者删除表时创建；
4. 字符锁：可以用`get_lock`等函数在服务器级别锁住/释放单个字符。

## 表锁
### 显式
```sql
lock tables film read;
lock tables film write;
unlock tables ...;
```
### 隐式
```sql
select sleep(30) from film limit 1
-- 相当于lock tables film read;
```
可以用`mysqladmin debug`命令检测锁的持有信息。（输出的末尾）

## 全局读锁
`show processlist`中`status`是`waiting for release of readlock`时，就是等待全局读锁了。

## 命名锁
`show processlist`中`status`是`waiting for table`时，就是等待命名锁了。
还可以在`show open tables`中看到命名锁的影响。

## 用户锁
```sql
-- 试图获得名为`my_lock`的锁，超时时间100秒。
select get_lock('my_lock',100);
```

## 使用information_schema查看锁
https://dev.mysql.com/doc/refman/5.6/en/innodb-information-schema-examples.html
哪个事务在等待锁，哪个事务持有锁：
```sql
SELECT
  r.trx_id waiting_trx_id,
  r.trx_mysql_thread_id waiting_thread,
  r.trx_query waiting_query,
  b.trx_id blocking_trx_id,
  b.trx_mysql_thread_id blocking_thread,
  b.trx_query blocking_query
FROM       information_schema.innodb_lock_waits w
INNER JOIN information_schema.innodb_trx b
  ON b.trx_id = w.blocking_trx_id
INNER JOIN information_schema.innodb_trx r
  ON r.trx_id = w.requesting_trx_id;
  
  
```

查看阻塞查询的线程元凶:
```sql
SELECT
	CONCAT('thread ', b.trx_mysql_thread_id, ' from ', p.host) AS who_blocks,
	IF(p.command = "Sleep", p.time, 0) AS idle_in_trx,
	MAX(TIMESTAMPDIFF(SECOND, r.trx_wait_started, CURRENT_TIMESTAMP)) AS max_wait_time,
	COUNT(*) AS num_waiters
   FROM INFORMATION_SCHEMA.INNODB_LOCK_WAITS AS w
   INNER JOIN INFORMATION_SCHEMA.INNODB_TRX AS b ON b.trx_id = w.blocking_trx_id
   INNER JOIN INFORMATION_SCHEMA.INNODB_TRX AS r ON r.trx_id = w.requesting_trx_id
   LEFT JOIN INFORMATION_SCHEMA.PROCESSLIST AS p ON p.id = b.trx_mysql_thread_id
   GROUP BY who_blocks ORDER BY num_waiters DESC;
   
+-------------------------+-------------+---------------+-------------+
| who_blocks              | idle_in_trx | max_wait_time | num_waiters |
+-------------------------+-------------+---------------+-------------+
| thread 4 from localhost |          98 |            12 |           1 |
+-------------------------+-------------+---------------+-------------+
```

结果看线程4空闲了98秒，至少有一个线程等了它12秒，有1个线程在等待它。