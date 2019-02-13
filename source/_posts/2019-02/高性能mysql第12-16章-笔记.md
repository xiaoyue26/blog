---
title: 高性能mysql第12-16章-笔记
date: 2019-02-13 09:34:54
tags:
- mysql
categories:
- mysql

---


这一章主要讲了一些概念和一些Mysql Cluster变种，略。
## 高可用概念
5个9: 99.999%，每年允许5分钟宕机时间。


## 宕机原因
1. 运行环境: 磁盘耗尽；
2. 性能问题：糟糕sql，schema索引设计;
3. 复制问题：主备不一致；
4. 数据丢失：如误操作。

### tips
1. 禁用查询缓存
2. 禁用复制过滤器、触发器。

# 高性能mysql第13章-笔记-云端mysql
IaaS:基础设施虚拟化；
DaaS:数据库虚拟化。

## Iaas:
mysql需要的4种资源:
1. cpu: 云cpu一般较慢；
2. 内存: 最好能到512GB~1TB；
3. IO： ms级，虚拟化后一般慢100倍； 
4. 网络： 一般不成为瓶颈。

## Daas：
稍微讲了一下亚马逊RDS，略过。

# 高性能mysql第14章-笔记-应用层优化
## 思考的角度
1. 应用和数据库正确分工；
2. ORM的循环SQL和嵌套查询的性能优劣；
3. 缓存(redis)；
4. 连接池。

## web服务器问题
静态资源： 用nginx代替apache
移除无用的apache模块

## 缓存问题
不同等级层次的缓存:
1. 进程缓存;
2. 本地共享内存缓存;
3. 分布式内存缓存;(memcached)
4. 磁盘缓存。

### 缓存失效策略
1. TTL;
2. 显式失效:更新数据时候更新缓存或者标记缓存为脏数据;
3. 读时失效:读的时候再判断缓存是否已经过期； （加快写，减慢了读）


# 高性能mysql第15章-笔记-备份与恢复
备份建议：
1. 使用Percona XtraBackup、Mysql Enterprise Backup；
2. 生产服务器和备份服务器分开，以免同时挂掉。
3. 定期进行恢复测试。

## 逻辑备份
(SQL,mysqldump)
缺点: 
1. 恢复起来慢

## 物理备份
(文件)
缺点：
1. 受版本、兼容性影响，bug概率高。
2. 文件很大。

建议：
0. 全局使用`utf-8`;
1. 逻辑：使用`mysqldump`备份结构；
2. 物理: 使用`select into outfile`导出数据为分隔符文件。
3. 恢复结构：执行1中的sql;
4. 恢复数据：`load data infile`。
其中2，4两步对字符集有要求。（不能单独设置某列字符集）

## 增量备份和全量备份
尽量做全备，增量bug多；

## 数据一致性
`mysqldump --single-transaction`
可能会导致非常长的事务，从而失败。

## binlog清理
```sql
purge master logs before current_date - Interval N day
```
或者设置`expire_logs_days`。


## innodb损坏恢复
### 1. 二级索引损坏
3种办法：
1. `optimize table`语句;
2. 删除重建表;
3. 表引擎改为`myisam`，再改回来。

### 2. 聚簇索引损坏
**优先使用备份还原**
否则：
(大概率只能修复未受损坏影响的行。)
通过`innodb_force_recover`选项导出表，如果导出过程崩溃，需要跳过受损行。

### 3. 损坏系统结构
损坏： innodb事务日志(redo log),表空间撤销日志(undo log),数据字典。

**优先使用备份还原**
否则：
可能需要做整个数据库的导出和还原。
innodb内部绝大部分工作可能受影响。

上述2，3两种损坏最好从备份还原数据。

# 高性能mysql第16章-笔记-用户工具
这章主要介绍一些工具：
### UI工具:
1. MysqlWorkbench;
2. SQLyog: 同workbench,windows专用；

### 命令行工具:
1. Percona Toolkit: 管理员必备；
2. The openark kit：一些管理任务的python脚本。

### 监控工具
1. Nagios;
2-6等其余很多略过。