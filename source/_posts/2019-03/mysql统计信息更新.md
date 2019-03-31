---
title: mysql统计信息更新
date: 2019-03-31 19:38:49
tags: mysql
categories:
- mysql

---

> 本文只关注innodb。 

mysql优化器选择执行计划的时候需要依据一定的采样统计信息，不然对数据完全不了解的话，就无法选择成本低的执行计划了。

统计信息的配置有以下几个自由度:
1. 是否持久化;
2. 更新统计信息的时机;
3. 采样多少个page。 

# 是否持久化
采样统计信息可以有两种选择：
1. 持久化: 默认是持久化，也就是存磁盘。
2. 非持久化.

控制的选项:
```
show variables like '%innodb_stats_persistent%';
```
默认是on，也就是持久化。
具体存哪里呢，主要是存mysql库和information_schema库下(5.6.x):
```
INFORMATION_SCHEMA.TABLES
INFORMATION_SCHEMA.STATISTICS
mysql.innodb_table_stats
mysql.innodb_index_stats
```

# 更新统计信息的时机
相关参数:
```
innodb_stats_on_metadata: 是否每次都重新计算统计信息(配合非持久化使用),默认off;
innodb_stats_auto_recalc: 插入数据量超过原表10%的时候更新统计信息,默认on。 
```
总结一下mysql更新统计信息的时机:
1. 手动运行触发语句如`analyze table xx`的时候;
2. 如果`innodb_stats_auto_recalc`为on: 插入数据量超过原表10%的时候更新统计信息;
3. 如果`innodb_stats_on_metadata`为on: 每次查询schema.table表的是更新统计信息(一般不开启，性能太差)。

# 采样page数量
相关参数:
```
| innodb_stats_sample_pages            | 8     |
| innodb_stats_persistent_sample_pages | 20    |
| innodb_stats_transient_sample_pages  | 8     |
```
`innodb_stats_sample_pages`废弃改成了`innodb_stats_persistent_sample_pages`和`innodb_stats_transient_sample_pages`,灵活控制持久化和非持久化下的采样page数。

可以看出默认情况持久化采样20个page。 

## 单表配置
上述所有都是全局配置，还可以为每个表单独3个参数:
```
STATS_PERSISTENT  : 1: 持久化统计信息;
STATS_AUTO_RECALC : 超过10%更新统计信息。
STATS_SAMPLE_PAGES: 采样页数。
```
可以看出为每个表设置的参数依然是这3个自由度: 是否持久化、更新统计信息时机、采样页数。