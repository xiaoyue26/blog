---
title: 高性能mysql第九/十章-笔记-选硬件/主备复制
date: 2019-02-11 22:41:01
tags:
- mysql
categories:
- mysql

---

这章主要讲怎么挑mysql的硬件。

# CPU
多核的

# 磁盘
raid卡，带电，防断电的


## iostat和vmstat用法
略

# 高性能mysql第十章-笔记-复制

使用场景：
1. 数据分布；
2. 负载均衡（从库可以读）；
3. 备份；
4. 高可用/故障切换；
5. 升级测试。 
{% img /images/2019-02/relay_log.png 800 1200 relay_log %}
主库备库同步流程：
1. 主库: 生成binlog;(binlog dump线程)
2. 备库: 读取binlong放入中继日志relay log;（通过TCP/IP协议,IO线程）
3. 备库: 重放relay log。(单线程,SQL线程)

## 配置复制
1. 每台服务器上创建账号;
2. 配置主库和备库；
3. 通知备库连接到主库，并从主库复制数据。

### 第一步创建账号
主库、备库：
```sql
grant replication slave, replication client on *.*
to repl@'192.168.0.%' indentified by 'p4ssword',;
```

### 第二步设置主库和备库
主库:
`[my.conf]`:
```conf
log_bin = mysql-bin
server_id = 10
```
重启主库。
检查主库完成配置:
```sql
show master status;

+------------------+-----------+--------------+------------------+-------------------+
| File             | Position  | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+------------------+-----------+--------------+------------------+-------------------+
| mysql-bin.000611 | 618598987 |              |                  |                   |
+------------------+-----------+--------------+------------------+-------------------+


```


备库配置：
`[my.conf]`
```conf
log_bin = mysql-bin
server_id = 2 
relay_log = /var/lib/mysql/mysql-relay-bin
log_slave_updates = 1 # 备库自身的重放事件也记录到binlog中
read_only = 1
```

### 第三步：启动复制
备库执行sql:
```sql
change master to MASTER_HOST= 'server1'
,MASTER_USER='reply'
,MASTER_PASSWORD='p4ssword'
,MASTER_LOG_FILE='mysql-bin.000001'
,MASTER_LOG_POS=0;
```
检查备库配置:
```sql
show slave status \G
```

让备库开始复制:
```sql
start slave;
```



## binlog配置
### 主库配置
`sync_binlog`=1: 事务提交前把binlog存到磁盘上。


## 复制的原理
`binlog_format`:
1. 基于语句的复制(逻辑): 出错机率大,开销小;
2. 基于行的复制(物理): 几乎不出错,开销一般更大。(5.1以后开始有)

基于语句的复制的**缺点**:
1. 不支持触发器或者存储过程：有bug;
2. 只能串行重放。

基于行的**缺点**:
1. 可读性差，无法直接知道执行了什么SQL。
解决方案: 
```sql
-- 1:
show master logs;
-- 2:
show binlog events in 'mysql-bin.000643' from 1073742091;
```


## 复制的过滤器
不建议使用。

## 复制的拓扑
### 1. 一主多备(星形模型)
### 2. 双主
不建议使用,两个主库都可写会有很多问题
### 3. HA模式(类似于双主,但只有一个可写)
{% img /images/2019-02/HA.png 800 1200 HA %}
两个库的配置基本相同，只是其中一个库是只读的。
如果有Alter table等耗时操作，可以在备库上执行，然后互换两个库。
优点：
故障恢复很简单。

具体设置情况:
1. 两个库数据相同;
2. 创建复制账号，选择不同serverid；
3. 两者启用binlog,互相跟踪;
4. 备库设置成只读，主库设置成可写。

由于serverid，主库不会重复消费自己的变更。（忽略自己的日志）

### 4. 环形(>=3的库成环)
不建议使用，太脆弱容易死循环。
### 5. 分发主库
{% img /images/2019-02/fenfa_mysql.png 800 1200 fenfa_mysql %}
一主多备对于备的数量有上限，可以用分发主库来进行扩展。
为了避免在分发主库上执行查询，可以将它的表修改为blackhole引擎。
（storage_engine=blackhole）

### 6. 模拟多主库
{% img /images/2019-02/moni_multi.png 800 1200 moni_multi %}
备库可以轮流读两个主库，俩主库带blackhole即可。

## tips
```
insert ...
select ...
-- 转换成:
select into outfile .. 
load data infile  ... 
```
这样更快(不需要加锁)


