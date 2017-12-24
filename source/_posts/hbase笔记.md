---
title: hbase笔记
date: 2016-01-08 19:23:46
tags: hbase
categories:
- hadoop
- hbase
---

http://hbase.apache.org/1.1/book.html
http://blog.csdn.net/u010270403/article/details/51648462

- 命令
```
-- cli启动
hbase shell
-- 列出表:
list
list 'pipe.*' -- 可以使用正则,列出pipe开头的表.
-- 建表mytable,族名是cf:
create 'mytable','cf' 
-- 写数据:
put 'mytable', 'first','cf:message','hello HBase'
# 在mytable表的插入rowkey为first的数据,列族为cf,列名message,具体数据为hello HBase.
put 'mytable', 'second','cf:foo',0x0
put 'mytable', 'third','cf:bar',3.15159
put 'mytable', 'forth','cf:kk','kk',3.15159 
-- 最后一个kk的3.15159被识别成timestamp写入了.
-- 看得出出来字符串需要单引号,数字则不需要
-- 不加字符串,不会被识别为表名或者列名,会被识别为函数名

-- 读数据:
get 'mytable','first'
-- 读整表:
scan 'mytable'
-- 输出:
hbase(main):031:0> scan 'MYTABLE'
ROW                             COLUMN+CELL
 112233bbbcccc                  column=NEWCF:attr, timestamp=1497410919789, value=data
 
-- 表名不一定大写,行键112233bbbcccc,列族为NEWCF,列名(qualifier)为attr,值为data.

-- 删除指定数据 
delete ‘scores','Jim','grade' 
delete ‘scores','Jim' 
-- 删除整行:
deleteall ‘scores','Jim'
-- 删除整个表: 
truncate ‘scores'
-- 更改表结构:
disable ‘scores' 
alter ‘scores',NAME=>'info' 
enable ‘scores' 
-- 增加列族名  
alter 'zmtest1', NAME => 'cf'
-- 删除一个列族
hbase> alter ‘t1′, NAME => ‘f1′, METHOD => ‘delete' 
hbase> alter ‘t1′, ‘delete' => ‘f1′ 

-- 执行脚本
hbase shell test.hbaseshell 


-- 



```

>HBASE写数据先写Memstore和WAL,最后由Memstore生成HFile.
每个列族可能有多个HFile,但每个HFile只能属于一个列族.
不清楚为什么WAL在Memstore前. 可能因为MemStore需要rpc,比较慢吧.
写操作生成:
1. 数据= Memstore+HFile(StoreFile)
2. 日志= WAL (HLog)


>读操作使用LRU算法,缓存名为BlockCache,与Memstore在一个JVM中.
HFile的物理存放形式是一个Block的序列外加这些Block的索引.
读路径包括:BlockCache,Memstore,HFile,将缓存的\未缓存未写入的\已写入的,三部分数据凑在一起返回客户端.

#表设计:
> Row key行键 (Row key)可以是任意字符串(最大长度是 64KB，实际应用中长度一般为 10-100bytes)，在hbase内部，row key保存为字节数组。
存储时，数据按照Row key的字典序(byte order)排序存储。设计key时，要充分排序存储这个特性，将经常一起读取的行存储放到一起。(位置相关性)
注意：
字典序对int排序的结果是1,10,100,11,12,13,14,15,16,17,18,19,2,20,21,…,9,91,92,93,94,95,96,97,98,99。要保持整形的自然序，行键必须用0作左填充。
列名都以列族作为前缀,例如courses:history ， courses:math 都属于 courses 这个列族.


# python api:happybase
首先在master上启动thrfit接口,注意`happybase`只能访问thrift接口,因此不是start thrift2:
```
./hbase-daemon.sh start thrift
```

python安装库:
```
pip install happybase
```
python代码:
```python
#!/usr/bin/env python
# encoding: utf-8
import sys
import traceback
from happybase import Connection

reload(sys)
sys.setdefaultencoding('utf8')

c = Connection('master')
t = c.table('MYTABLE')
count = 0
for _ in t.scan(filter='FirstKeyOnlyFilter() AND KeyOnlyFilter()'):#filter='FirstKeyOnlyFilter() AND KeyOnlyFilter()'
    count += 1

print count
```

#建立结构相同的表
`habse`没有传统数据库中类似`create table like`的语句.
如果要建立一个与现有表结构相同的表,只能使用如下步骤:(假设要达到的目的是`create table 'pipe:zeb_log' like 'pipe_day'`)
``` 
// 1. 获取原表结构:
desc 'pipe_day'
// 2.
把从{name=>...}开始的部分做如下处理:
(1)   TTL => 'FOREVER'   替换成  TTL => org.apache.hadoop.hbase.HConstants::FOREVER
(2)   {name=>...} 之间加上逗号
(3)   整条语句变成一行,句首加上 create 'pipe:zeb_log'

// 3. 
```

# HBase导出数据
1. 导出成`SequenceFile`:// 没啥用,无法解析.
```
 hbase org.apache.hadoop.hbase.mapreduce.Export pipe:zeb_log /user/fengmq01/zeb_log
# 把 pipe:zeb_log 导出到了 /user/fengmq01/zeb_log 文件夹
# 命令格式:
# hbase org.apache.hadoop.hbase.mapreduce.Export <tablename> <outputdir> [<versions> [<starttime> [<endtime>]]]
```

# 数据结构
使用上可以简单得想象成一个树形结构:
Table(Rows) --通过row key 索引到=> Row ;
Row         --通过column family索引到CF;
CF -- 通过column qualifier 索引到最新版本的Value. 
不同timestamp的value组成Cell. 
```
Table(Rows)->CF->Cell->Value. 
```
每一级都是一个KV存取. 