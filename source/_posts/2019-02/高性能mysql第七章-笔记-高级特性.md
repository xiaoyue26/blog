---
title: 高性能mysql第七章-笔记-高级特性
date: 2019-02-11 22:31:10
tags:
- mysql
categories:
- mysql

---

# 高级特性
略过: 分区表,合并表,全文索引

## 视图
视图会影响优化器

## 存储过程
在线系统: 不推荐使用;
离线系统: 如果代码很简单，为了批量性能可以考虑。

## 游标
游标使用临时表实现。

因此：
1. 逐行只读;
2. 会执行整个查询。

基于2，如果只读一小部分结果，请使用limit。
(也就是不要使用游标来实现limit的效果，直接使用limit语句即可)

## prepareStatement
### 流程
1. 客户端=>服务端: SQL原型
2. 服务端: SQL原型=>部分执行计划A;
3. 服务端=>客户端: A的句柄;
4. 客户端=>服务端: 变量、A的句柄.

优点:
1. 服务端: SQL解析只需解析/优化一次;
2. 通信: 只发送参数+句柄，通信成本降低;
3. 缓存: 参数可以被缓存;
4. 安全: 无须转义、由于执行计划定了，不会被SQL注入。

`prepareStatement`的三类优化:
(和传入参数无关的)
1. 准备阶段: 根据已知条件,where条件优化;
2. 第一次执行: 简化嵌套关联,将外关联转化成内关联;
3. 每次SQL执行:
(1) 过滤分区;
(2) 尽量移除count,min,max;
(3) 移除常量表达式;
(4) 检测常量表;
(5) 做必要的等值传播;
(6) 分析和优化ref,range和索引优化等访问数据的方法;
(7) 优化关联顺序。

## 绑定变量
`set @xxx:= 123`
绑定变量是会话级的。

## 字符集和校对
字符集(`encode`)： 二进制<=>某类编码字符
校对(`collation`)：某字符集的排序规则。

校对集都是针对某一个字符集的。(类似于弱实体)

相关设置分为两类:
1. 创建对象时候的默认值
2. 服务器和客户端通信的设置

### 1. 创建对象时候的默认值
(1)`character_set_server`: 服务器创建数据库默认值;
(2)表字符集;
(3)列字符集.
越小范围的优先级越高。


### 2. 服务器和客户端通信的设置
#### 服务端:
{% img /images/2019-02/mysql_char.png 800 1200 mysql_char %}
(1)`character_set_client`: 服务端总是假设客户端按照`character_set_client`设置的字符集来传输数据和sql语句;
(2)`character_set_connection`: 服务器收到客户端sql后,转换成`character_set_connection`类型;
(3)`character_result`:服务端返回数据时,转换成`character_result`类型。

#### 客户端:
客户端: `jdbc:mysql://localhost:3306/exceltest1?useUnicode=true&characterEncoding=UTF-8`

客户端可以使用`set names utf-8`语句来告诉服务端自己将使用`utf-8`传输数据。

#### 特殊情况
诡异的`character_set_database`: 某个数据库的默认字符集。当数据库改变的时候，它也会改变。当没有指定数据库的时候，会按照`character_set_server`。

`Load Data Infile`:
数据库总是将文件中的字符按照字符集`character_set_database`来解析。
mysql加载数据的时候，总是以同一个字符集处理所有数据，不管表中的列是否有不同的字符集设定。

`Select Into OutFile`:
输出文件时不会自动做转码，也不能指定。唯一的方法是手动调用函数`convert`。

#### 查看字符集设置
```sql
SHOW VARIABLES LIKE 'character%';
```

查看mysql支持的校对集: `show collation`

一般校对集是字符集加上三种后缀:
`_bin`: 二进制比较;
`_ci`: 忽略大小写(ignore);
`_cs`: 大小写敏感(sensitive)

大小写敏感比二进制比较更加复杂，会有更多规则。

### mysql如何选择字符集
每个字符集有对应的默认校对集；
每个校对集有对应的默认字符集。（一般来说，校对集依赖于字符集）
1. 如果用户设置了: 那肯定按用户设置的来;
2. 如果设置了一部分: 根据用户设置找默认的。
3. 什么都没设置: 根据默认配置设置。

# 分布式（XA）事务
两阶段提交。
协调者+参与者。
1. 所有参与者完成准备工作；
2. 协调者收到所有参与者的回复，提交事务。

两种XA: 内部XA,外部XA。

## 内部XA
mysql是插件式的，因此每个存储引擎是互相独立，不知道彼此存在的。
他们之间协调需要分布式事务。(内部XA)

- 跨存储引擎的事务需要内部XA。

如果开启了二进制日志，日志也可以看作一种特别的存储引擎，因此也需要内部XA。

## 外部XA
mysql对xa协议支持不完整，无法在一个XA中关联多个连接。
外部XA网络开销大，建议引入`MQ`解决，而不使用外部XA。

# 查询缓存

1. 检查sql是不是`select`开头,如果数据源表没有更改,hash查找缓存,返回结果;
2. 如果有确定性结果,存储查询语句的结果和hash值到缓存。

由于2(确定性结果)，所以使用`current_date`这样的函数时，sql结果不会被缓存。

查询缓存的注意事项:
1. 不能有长时间的事务;(会引入全局锁)
2. 不要占用太大内存(过度依赖缓存)，可以直接使用redis。

过度依赖缓存的话，缓存清理以后系统会假死很久。

## 查询缓存的配置参数
`show variables like '%query_cache%'`
### query_cache_type
三种取值:
1. OFF(0): 关闭,默认值。
2. ON(1):  打开
3. DEMAND(2): sql里写明`sql_cache`hint的语句才放入缓存。

`query_cache_size`: 查询缓存占的空间;
`query_cache_limit`: 缓存结果集最大条数;

清空缓存: `reset query cache`
整理缓存碎片: `flush query cache`(服务僵死一段时间)
{% img /images/2019-02/query_cache_opt.png 800 1200 query_cache_opt %}