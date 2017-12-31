---
title: sqoop的SQL注入漏洞
date: 2017-12-31 21:29:38
tags: sqoop sql注入
categories:
- hadoop 
- sqoop

---


早上起来发现公司有一个sqoop导表任务挂了,查看日志错误提示大致如下:
```
INFO [main] org.apache.sqoop.mapreduce.db.DBRecordReader: Executing query: SELECT id,xxx FROM taname where ( udid >= 'abc' ) AND ( udid < 'xx' ;select pg_sleep(3) --)
ERROR 1064 (42000): You have an error in your SQL syntax; check the manual that corresponds to your MySQL server version for the right syntax to use near ')' at line 1
```

也就是说sqoop生成的查询语句错了,搜了一下源码没有发现哪里会生成
```
select pg_sleep(3)
```
这种语句,而且我们用的是mysql,按理说不应当生成pgsql.
而且这个导表任务也不是第一天上线了,此前一直是好好的.

首先按照经验,调整了分隔符配置;查看表结构发现有blob字段,去掉了配置中的direct模式,均没有效果.
其次查job的其他日志,发现其他几个mapper都成功了,只有1个mapper挂了,于是调整并行度,把并行度调成1,导表任务就成功了.

回过头来继续研究这个问题.
也就是说在分割表数据到几个mapper的时候,划分split的时候,查询语句错了.
报错的上一行的日志大致是:
```
org.apache.hadoop.mapred.MapTask: Processing split: udid >= 'xxx' AND udid < 'xxx'); select pg_sleep(3) -- '
```

sqoop划分split的原理:
1. `--table` 按表导: 按主键最大最小取值划分范围,然后按中间值划分split.
2. `--query` : 按配置中的`-- split by`参数对应的列取max,min,然后同上.

因此如果作为划分列的值中如果有脏数据,sqoop就会被sql注入.
因此导致了我们遇到的问题.
平时脏数据没有落在分割点上, 今天可能是由于脏数据的比例逐渐上升,终于落在了分割点上,因此导致了任务失败.
(`pg_sleep`一般是用于sql注入.)