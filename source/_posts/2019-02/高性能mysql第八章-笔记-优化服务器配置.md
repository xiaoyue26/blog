---
title: 高性能mysql第八章-笔记-优化服务器配置
date: 2019-02-11 22:37:32
tags:
- mysql
categories:
- mysql

---

> 错误配置不如维持默认配置。

# 优化服务器配置
## 配置文件
服务端读取`mysqld`分段
客户端可能会读取客户端分段。

## 配置级别
全局、连接级、会话级
`sort_buffer_size`: 每个线程都可以设置
`join_buffer_size`: 可以为每个关联都设置一个关联缓冲。

## tips
用git管理配置文件

## 最优配置
书上也不知道怎么配出最优配置。
1. 用benchmark：穷举配置成本太高；
2. 用比较低的值慢慢试探，较低的内存配置对性能影响不大（百分之几），但较高的内存配置会造成较大的性能抖动，因此尽量以较低的配置。

## Buffer pool（Innodb）
缓存内容:
1. 缓存索引;
2. 缓存行数据；
3. 缓存自适应哈希索引；
4. 插入缓冲；（延迟合并写入）
5. 锁。

## Key caches(MyIsam)
键缓冲(键缓存)。
MyIsam只缓存索引，不缓存数据。

查看MyIsam的索引总大小:
```sql
select sum(index_length) 
from information_schema.tables
where engine='MYISAM';
-- 也可以看看分布:
select engine,sum(index_length) 
from information_schema.tables
GROUP BY engine;

+--------------------+-------------------+
| engine             | sum(index_length) |
+--------------------+-------------------+
| CSV                |                 0 |
| InnoDB             |       68176035840 |
| MEMORY             |                 0 |
| MyISAM             |            124928 |
| PERFORMANCE_SCHEMA |                 0 |
+--------------------+-------------------+

```
key cache的大小不要超过MYISAM索引总大小。


## 线程缓存
`thread_cache_size`:  缓存线程数量。

```sql
show variables like '%thread_cache_size%';
show variables like '%thread%';


+-----------------------------------------+---------------------------+
| Variable_name                           | Value                     |
+-----------------------------------------+---------------------------+
| innodb_purge_threads                    | 1                         |
| innodb_read_io_threads                  | 12                        |
| innodb_thread_concurrency               | 0                         |
| innodb_thread_sleep_delay               | 10000                     |
| innodb_write_io_threads                 | 12                        |
| max_delayed_threads                     | 20                        |
| max_insert_delayed_threads              | 20                        |
| myisam_repair_threads                   | 1                         |
| performance_schema_max_thread_classes   | 50                        |
| performance_schema_max_thread_instances | -1                        |
| pseudo_thread_id                        | 4804578                   |
| thread_cache_size                       | 512                       |
| thread_concurrency                      | 4                         |
| thread_handling                         | one-thread-per-connection |
| thread_stack                            | 524288                    |
+-----------------------------------------+---------------------------+
```

## 表缓存
对myisam有用，innodb用处不大。
算是一个已经过时的配置。

## Buffer Pool相关架构
{% img /images/2019-02/buffer_pool.png 800 1200 buffer_pool %}
图中的undo日志是方便事务回滚的。
(redo日志是类似WAL，防断电的，利用顺序IO优化性能)

事务日志(redo log)使用类似环形数组的数据结构，可以环形写入到flush过的头部，节省存储空间。


### Log Buffer（Redo log/事务日志）
每次变更: 记录到Log Buffer(Buffer Pool => Log Buffer)

Log Buffer刷盘到磁盘日志文件的时机(或者说触发事件):
1. 每一秒钟(定时器tick事件);
2. 或事务提交事件;
3. 或缓冲区满。

以上是默认的三个事件，调节参数为`innodb_flush_log_at_trx_commit`,3种取值:
0: 事务提交:不触发;
1: 事务提交:刷盘到磁盘;(默认值，3种事件皆触发刷盘)
2: 事务提交:刷新到操作系统缓存。（能扛mysql进程挂掉，不能扛断电，断电的话会丢一秒数据。）


如果缓冲区满的太频繁(1s满了很多次)，可以考虑增大缓冲区大小来减少IO次数。
参数:
`innodb_log_buffer_size`: 默认1M，推荐1~8M。

调节这个参数需要命令:
```sql
show engine innodb status
```

## Innodb读写日志、数据文件配置
这是一个风险很高的选项，认真学习以后有必要再修改。
```sql
show global variables like '%flush_method%';

+---------------------+----------+
| Variable_name       | Value    |
+---------------------+----------+
| innodb_flush_method | O_DIRECT |
+---------------------+----------+
```
windows用的选项:
async_unbuffered,unbuffered,normal.

linux：

- `fdatasync`: 默认值。用`fsync()`刷新数据和日志文件。
`fdatasync()`函数只刷新文件的数据；
`fsync()`函数刷新数据和日志文件。
配置为`fdatasync`时其实会调用`fsync()`函数。默认会合并多个文件的fsync()操作，合并IO操作。可以通过`innodb_file_per_table`选项为每个文件独立fsync()，但也会增多IO操作。

- `O_DIRECT`: 数据文件: 用`O_DIRECT`标记或`directio()`函数。
（不影响日志文件）
区别: 使用fsync()函数刷盘，但会通知操作系统不要缓存数据，也不要预读。
相当于关闭了操作系统缓存（读写）。
特例： 不影响raid卡的预读、写缓存。

- `ALL_O_DIRECT`: 影响数据和日志文件。

- `O_DSYNC`: 日志文件: 用`O_SYNC`标记，写同步（日志写到磁盘才返回）
（不影响数据文件）
区别：不禁用操作系统的缓存。

| 配置值 | 实际调用函数 | 概述 |
| :--: | :--:|:--:|
| fdatasync | fsync(含义相反)| 刷新数据和日志，有操作系统缓存 |
| O_DIRECT|   | 刷新数据，关闭缓存 |
| ALL_O_DIRECT| | 刷新数据和日志，关闭缓存 |
| O_DSYNC| O_SYNC | 刷新日志，有操作系统缓存 | 

O_DSYNC和fdatasync的区别:
（实际是O_SYNC和fsync的区别）fsync允许写操作累积在缓存，然后一次性刷新。
O_SYNC是每次都是同步IO，每次都刷盘。

### tips
如果使用raid卡带电: O_DIRECT;
否则: O_DIRECT或fdatasync都有可能，看业务需求。


## Innodb表空间
位于`innodb_data_home_dir`目录:
```
表、索引、回滚日志(undo log)、插入缓冲(Insert Buffer)、双写缓冲(DoubleWrite Buffer)。
```

目录下可以挂载多个设备，但不会自动做负载均衡，只有写满第一个才会写第二个，因此需要底层用raid自己做负载均衡。
挂载多个目录:
```
innodb_data_file_path = /disk1/ibdata1:1G;/disk2/ibdata2:1G;/disk3/ibdata3:1G:autoextended:max:2G
```

建议使用`innodb_file_per_table`选项，每个表的表空间一个文件。
缺点：DROP Table慢。
原因: 
1. 删除文件慢；
2. 移除表空间需要Innodb锁定和扫描缓冲区，查找属于这个表空间的页。

#### 运行时间长的事务
缺点: undo log增长到打爆表空间、磁盘。

### 双写缓冲(Doublewrite Buffer)
功能: 避免页没写完整导致的数据损坏。
buffer pool => Doublewrite Buffer => 磁盘

写两次，但由于有`fsync()`，而且这个过程时间顺序上比较紧密，可以一次性刷到磁盘，写两次的性能损耗也就多了百分之记。

### binlog IO配置
`sync_binlog`: binlog配置。
N=0(默认值): 不干预，让操作系统自己决定。
N>0: N次写以后，刷新到磁盘。（一般不会大于1)
```
show global variables like '%log%';
```

innodb的6种日志:
1. redo log(物理/WAL): 事务开始后写入缓冲区innodb_log_buffer;（存储引擎层）
2. undo log(逻辑/版本): 事务开始后为了事务回滚准备的日志，从buffer pool=>表空间;
3. bin log(逻辑/SQL): 主从复制。或者基于时间点的还原。(类似于存了sql)。（事务提交前产生，数据库层）
4. slow query log: 慢查询;
5. general log: 一般日志;
6. relay log: 中继日志。


undo log（在表空间）: 修改前的值/版本号;
undo log可以配置从表空间中分离出来。

redo log（在log buffer）: 类似于WAL，记录buffer pool和disk的不同。

### 回滚事务的crash恢复
- redo log: 只追加写，提交到磁盘以后标记为可以被新日志覆盖。
- 基于这个特性，即使事务已经回滚了，它的redo log依然是存在的。
- 再基于以上这点，db crash以后，可以恢复已经回滚的事务。

回滚事务的crash恢复,有两种策略:
1. 读redo log,挑已经提交的事务进行恢复;
2. 读redo log,恢复所有事务，再用undo log解决回滚的事务。

`innodb`使用的是策略2，也就是先redo,再undo。

使用策略2的难点:

- 需要redo log相应的undo log是健全的，否则就漏回滚了。
(1)解决方案1:确保undo log落盘早于redo log；（很麻烦，不用）
(2)解决方案2:将undo log也视为数据，记录到redo log中。（innodb采用）
 
- 总结
> crash恢复: 先读取执行redo log，再执行undo log，这样可以恢复未提交事务和回滚事务。
undo log和redo log的数据一致性: 把undo log也视为数据，写入redo log。


### 日志与事务提交速度:
redo log: 不影响事务提交速度。因为是事务进行中就产生、而且每秒刷盘了；
binlog: 影响事务提交速度。因为事务提交前才产生binlog、刷盘，因此如果开启了binlog，长时间的事务提交很慢。

三种重要日志的写顺序:
1. undo log;
2. redo log;
3. binlog.

## MYISAM的IO配置
`delay_key_write`:
OFF: 不延迟。每次写操作后刷新键缓存; 
ON: 延迟。对使用该选项创建的表生效;
ALL: 延迟。对所有表生效。
该选项对性能提升不大,缺点:
1. 索引可能损坏;
2. 关闭表时间变长;
3. 占用空间变大。

`myisam_use_mmap`: 使用内存映射，减少系统调用开销。

## Innodb并发
`innodb_thread_concurrency`:
N=0: 不限制有多少线程进入内核;
N>0: 同时可以有N个线程进入内核。
N的推荐值< 2*cpu数*disk数

`innodb_commit_concurrency`:
同时可以有多少个线程提交。

## Myisam并发
`concurrent_insert`:
0: 不允许并发插入;
1: 默认值。只要表中没有空洞，就允许并发插入。
2: 强制并发插入到Myisam表末尾，即使有空洞。（碎片会更多）

## 基于工作负载的配置
1. blob,text的表，临时表都会变成磁盘表。
解决方案: 使用substr。

`tmp_table_size`,`max_heap_table_size`: 临时表大小上限。
`max_connections`: 最大连接数。
`thread_cache_size`: 缓存线程数。

 查看配置和状态:
 ```
 show variables like '%thread%';
 show status like '%thread%';
 ```
 
 
 `innodb_autoinc_lock_mode`
 0: 每次都锁；
 1: 确定需要分配多少自增值的话，可以立即释放锁；
 2: 每次都不锁，空洞很多。
 
## 两个最重要的配置
 `innodb_log_file_size`: redo log的缓冲区，如果调大了`innodb_buffer_pool_size`就要相应调大它。
(一种可能的值是512*1024*1024,512M,搭配2G的`innodb_buffer_pool_size`)
可以通过`show engine innodb status\G select sleep(60); show engine innodb status\G`,查看`Log sequence number`差，计算1个小时的redo日志写入量，大致可以设置为这个缓冲区的大小。


`innodb_buffer_pool_size`的大小设置就比较简单了，按服务器可用内存75%设定即可。



