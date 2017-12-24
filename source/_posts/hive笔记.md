---
title: hive笔记
date: 2017-01-22 19:34:41
tags: hive
categories:
- hadoop
- hive

---

# hive 小技巧
GROUP BY xxx WITH CUBE的时候,要区分是维度是total_count还是null,
可以用GROUPING__ID.
当GROUPING__ID的二进制在某列为0,则为total_count,否则是具体值导致的null. (换句话说就是需要二进制操作,代码会很复杂,需要udf)

# hive 1.2.1 bug:
```
1. alter table temp.feng_test1  add COLUMNs (col2 string);
增加一列后,无法读取新的一列的数据.

现象: 使用alter语句增加一列,重新insert overwrite数据后,新增列的数据始终为null.

解决方案: 
1. 对于外部表,可以通过重建整个表结构解决.
首先保存建表语句,然后drop table,最后重建整个表即可.

2. 对于有分区的内部表,可以通过重建分区解决,首先使用drop partition语句删除分区和数据,然后重跑这个分区的数据(或者事先备份好数据然后add partition). 注意,区别于直接使用insert overwrite partition语句.
3. 对于无分区的内部表,暂无特别好的办法,只能先把表数据备份到另一个目录,(备份表结构和数据)然后drop table,最后重建表即可.注意,区别于直接使用insert overwrite table语句. 
```

# hive传参数:
```
hive -d DATE=1 -e 'select ${DATE}'
```

# hive 更新函数:
> 注意事项: DROP function xxx时,得保证创建函数时用的jar还在,不然可能导致更新函数不成功,每次运行时还是会加载原有的jar包.
```
SELECT thriftparser(answerresults, "com.fenbi.ape.hive.serde.thriftobject.conan.AnswerResults")
FROM ape.ori_mysql_ape_conan_task_task_report_da
limit 10
;

drop function thriftparser;

create function thriftparser AS 'com.fenbi.ape.hive.serde.ThriftSerdeUniversalUDF' USING JAR 'hdfs://f04/lib/ape-serde-1.1-SNAPSHOT.jar';

reload function;
reload functions;
```


从8088获取hive查询的详细语句：
http://f04:8088/proxy/application_1465276959835_417313/mapreduce/conf/job_1465276959835_417313
具体操作方法是先进入job的ApplicationMaster，具体jobid链接，左侧configuration，然后在右侧下方小字的key输入框，输入hive.query.string进行查询即可。


突然想到，对数据量小的表可以先做Distinct,数据量大的表可能得用group by。
前者用一个reduce即可，后者可以用多个reduce,跑得快一点。
http://idea.lanyus.com/
需要看的教程:
[https://issues.apache.org/jira/browse/HIVE-591]
[http://git-scm.com/book/zh/v2][1]
[http://pcottle.github.io/learnGitBranching/?demo][2]
`smartGit`
`derby`数据库是什么? 就是一个轻量级的数据库 好多apache项目都默认自带。

查看所有函数/查看指定函数用法。
```
SHOW FUNCTIONS; 
desc function find_in_set;
```

- `sql code style flow`:
```sql
SELECT
FROM
WHERE
GROUP
HAVING
```

`hbase`是直接查询(KV)；
`hive`会转化为`map-reduce`任务。
`pig`是定制`reduce`部分。

`hive`使用列式存储，使用`dt=?`指定`partition`会加快查询速度，像索引一样。[列式存储基础知识][3]
![列式存储天生适合压缩][4]
- 列式存储天生适合压缩(相同数据类型)
- 列式存储数据库对分布式支持友好
 

![投影等操作的对比][5]

- 自然连接是一种去重复列后的等值连接，它要求两个关系中进行比较的分量必须是相同的属性组，并且在结果中把重复的属性列去掉。而等值连接并不去掉重复的属性列。
- 选择->限制->`where`—>指定行;投影->`select`->指定列。
 
列式存储查询流程:
![列式存储查询流程][6]
1.     根据`where`的行限制,去字典表里找到字符串对应数字(只进行一次字符串比较)。
2.     用数字去列表里匹配，匹配上的位置设为1。
3.     把不同列的匹配结果进行位运算得到符合所有条件的记录下标。
4.     使用这个下标，再根据`select`语句中指定的列,组装出最终的结果集。


---

`$HOME/.hiverc`目录下可设定启动脚本。
如`set hive.metastore.warehouse.dir/=...`

查看`hive`版本：
```shell
hive>set hive.hwi.war.file;
$ echo $HIVE_HOME
/home/maintain/hive/apache-hive-0.14.0-bin

#终止:
ctrl+c (不是win+c)
两次。
若未执行完毕则终止的是jvm;
若执行完毕则返回结果在文件系统里，切断的是与文件系统的联系。
```

设定变量:
```shell
set foo=bar2;
set hivevar:foo=bar2; #等效
set foo;# 查看结果
```

`hive`中可使用`shell`命令等:（也是分号结尾）
```shell
! /bin/echo "hello world";
! pwd
dfs -ls / ;
-- copyright ... #注释以`--`开头
```

注释以`--`开头 比较特殊。

各种分隔符`p46`。

一个概念：`读时模式`：
传统数据库是`写时模式`，在写入的时候进行模式检查、有效性验证。
`hive`是`读时模式`，就是加载数据的时候才进行验证，尽可能恢复数据的有效性和合法性。对于错误的类型返回`null`,如数值类型中存放了字符串。

`hive`不支持行级操作、不支持事务。

可以重复使用`Use`指令，`hive`没有嵌套数据库。

递归删除数据库
`drop database if exists xxxbase cascade;`

支持正则:(`show databases`里用不了)
`show tables 'empl.*';`
mysql 里是:`show tables like '%empl%' ;`

输出表信息:
`describe formatted mydb.employees;`


`set hive.stats.reliable=false`是什么含义，设定为暂时不可用?
这个是把状态收集器关掉，效果是在insert overwrite的时候，不去重。

 
 查看是管理表还是外部表:
 `describe formatted mydb.employees;`
输出信息的中部有这一项(`table_type`).

创建相同表结构但没有数据的表:
```sql
create table if not exists mydb.employees3
Like mydb.employees
Location '/path/to/data';
```

创建带分区的表:
```sql
create table employees(
name STRING,
salary Float,
subordinates Array<String>,
deductions map<string,float>,
address struct<street:string,
               city:string,
               state:string,
               zip:int>
)
partitioned by (country string,state string);
```
`hive`会按照分区创建目录。然后不在字段中存储分区信息。

查看已有的分区设置:
```sql
show partitions tmp_tutor_user_profile;
```

设置查询时是否必须指定分区:
```sql
set hive.mapred.mode=strict;
set hive.mapred.mode=nonstrict;
```

载入数据的方式创建分区:`p60`

`hive`不储存建表语句?（`Navicat for MySQL`中的对象信息）。
`show create table test1` 里有.


增加分区信息：
```sql
alter table log_messages 
add partition
(year=2015,month=1,day=2)
location 'hdfs://master_server/data/log/2012/01/02';
```

`emailUtils`

- 大部分`alter`操作不会改变数据，只改变元数据。
```sql
alter table log_messages
partition(year=2012,month=12,day=2)
set location 's3n://ourbucket/logs/2011/01/02';
```
上述命令不会把数据从旧的路径转移或者删除。

---
- 例外：
```sql
alter table log_messages 
drop if exists partition
(year=2012,month=12,day=2);
```
上述语句会删除管理表中的数据，而不会删除外部表中的数据。

- 对字段进行重命名、修改位置、类型和注释：
```sql
alter table log_messages
change column hms hours_minutes_secends INT
comment 'the hours,minutes and seconds part of the timestamp'
after severity;
```

- 装载数据:
```sql
load data local inpath '${env:HOME}/california-employees'
overwrite into table employees
partition (country='US',state='CA');
```
如果分区目录不存在，命令会先创建分区目录然后再将目录装载(拷贝)到该目录下。`hive`会检查文件格式是否与表结构相符，但不会检查数据是否和表模式匹配。(读时模式)

从`hdfs`中装载数据时,不使用`local`关键字。

`p133`中? 存疑. `into`? `table`?
应该是into.
```sql
load data local inpath 'log2.txt' into weblogs 
partition(20110102);
```

-  动态分区
```sql
insert overwrite table employees
partition (country,state)
select ..., se.cnty, se.st
from staged_employees se;
```
混合动态和静态分区:
```sql
insert overwrite table employees
partition (country='US',state)
select ..., se.cnty, se.st
from staged_employees se
where se.cnty='US'
;
```
静态分区必须在动态分区前。

- 属性设定:
<table>
     <tr>
      <td>属性名称</td>
      <td>缺省值</td>
      <td>描述</td>
   </tr>
   <tr>
      <td>hive.exec.dynamic</td>
      <td>false</td>
      <td>设置true即开启动态分区功能</td>
   </tr>
   <tr>
      <td>hive.exec.dynamic.mode</td>
      <td>strict</td>
      <td>nonstrict表示运行所有分区都是动态的</td>
   </tr>
   <tr>
      <td>hive.exec.max.dynamic.partitions.pernode</td>
      <td>100</td>
      <td>每个mapper或reducer可以创建的最大动态分区个数</td>
   </tr>
   <tr>
      <td>hive.exec.max.dynamic.partitions</td>
      <td>+1000</td>
      <td>一个语句可以创建的最大动态分区个数</td>
   </tr>
   <tr>
      <td>hive.exec.max.created.files</td>
      <td>100000</td>
      <td>全局可以创建的最大文件个数</td>
   </tr>
</table>

- 导出数据
1. 格式相同:
```shell
hadoop fs -cp source_path target_path
```
2. 格式不同:
```sql
insert overwrite local directory 'tmp/ca_employees'
select name, salary, address
from employees
where se.state='CA'
;
```
3. 多输出目录:
```sql
from staged_employees se
    insert overwrite directory '/tmp/or_employees'
     select * from where se.cty='US' and se.st='OR'
    insert overwrite directory '/tmp/ca_employees'
     select * from where se.cty='US' and se.st='CA'
    insert overwrite directory '/tmp/il_employees'
     select * from where se.cty='US' and se.st='IL'
;     
```


- 查看结果文件内容:
```sql
! ls /tmp/ca_employees;
! cat /tmp/ca_employees/000000_0 ;
```

取整:` round (salary)`
数学函数`p82`
```java
byte int short long 1,2,4,8
boolean char float double 1,2,4,8
```

- 部分内置函数
<table>
     <tr>
      <td>返回值类型</td>
      <td>样式</td>
      <td>描述</td>
   </tr>
   <tr>
      <td>type</td>
      <td>cast(expr as type)</td>
      <td>把expr转换为type类型</td>
   </tr>
    <tr>
      <td>string</td>
      <td>concat(binary s1,binary s2,...)</td>
      <td>拼接字符串</td>
   </tr>
   <tr>
      <td>string</td>
      <td>concat_ws(string separator, string s1,string s2,...)</td>
      <td>使用指定分隔符拼接字符串</td>
   </tr>
</table>

`set hive.exec.mode.local.auto;`这个值为什么是`false`?

由于浮点数的不准确性，与钱有关或者涉及到比较的关键数字都不使用浮点数，使用`string`.

- 正则
 - `like`使用`sql`通配符(`%abc%` )
 - `Rlike`使用`java`正则表达式(`.*(abc).*`)

---

- 聚合函数、表连接:
```sql
select year(ymd), avg(price_close) from stocks
where exchange='NASDAQ' AND symbol='AAPL'
group by year(ymd)
having avg(price_close)>50.0
;
# 注释:'--'
-- inner join: 表应从小到大排列以便hive优化;
select * from a.ymd, b.price_close
from stocks a join stocks b on a.ymd=b.ymd
where a.symbol='APPL' 
AND b.symbol='IBM'
;

-- left outer join 左外连接
# 左表中符合where子句的所有记录将返回
# 右表中不符合on连接的列将返回NULL
select * from a.ymd, b.price_close
from stocks a left outer join dividend b on a.ymd=b.ymd
where a.symbol='APPL' 
;
-- 执行顺序, 先执行join,再使用where进行过滤。

-- 嵌套select:
select s.*, d.* from
(select * from stocks where ...) s
left outer join
(select * from dividend where...) d
on s.ymd=d.ymd
;

--类似的, 右外连接 right outer join
-- 全连接 full outer join

-- 左半开连接: left semi-join
select s.* from stocks s
left semi join
dividends d 
on s.ymd=d.ymd and s.symbol=d.symbol
;
#返回左边表的记录，用on中的条件进行过滤。
#比inner join(直接join)更高效，但不能返回右表中的数据。

-- 笛卡尔积: 没有on语句的join

```

- `map-side join`
通过设置`set hive.auto.convert.join=true;`开启。

- `distribute by`
`distribute by`控制`mapper`的输出在`reducer`中是如何划分的。
```sql
select * from stocks s
distribute by s.symbol
sort by s.symbol asc, s.ymd asc
;
```
此处`distribute by`指定具有相同股票交易码的记录会分发到同一个`reducer`中进行处理。`cluster by s.symbol`相当于`distribute by s.symbol sort by s.symbol desc`的简写,只支持降序。

`order by`保证全局有序;
`sort by`只保证每个`reducer`的任务局部有序。

- 抽样查询
```sql
select * from numbers 
tableSample(bucket 3 out 10 on rand())
s;
-- 指定列分桶:
select * from numbers 
tableSample(bucket 3 out 10 on number)
s;
```

- 数据块抽样(百分比)
```sql
select * from numbersflat TableSample(0.1 percent)
s;

-- 最小抽样单元是一个hdfs数据，所以数据大小小于128MB时会返回所有数据
-- 不一定适用于所有文件格式
```

哈斯分区是什么? `p115`
分隔符的可读性为何这么低还是不清楚。（?）
```sql
-- 两种顺序调换的意义?
from...insert...select...
from...select...
```


示例文件夹:
`/Users/xiaoyue26/Documents/pipe_warehouse/solar/dwSolarUserStat`

- 索引
```sql
create Index employees_index
on Table employees (country)
as 'org.apache.hadoop.hive.ql.index.compact.CompactIndexHandler'
with Deferred Rebuild #意思不要立刻开始创建索引的数据,只是先声明一个索引
-- 新索引呈空白状态
IDXproperties ('creator'='me', 'created_at'='some_time')
in table employees_index_table #可选 也可以把索引建成一个文件
partitioned by (country, name)
comment 'Employees indexed by country and name.'
;
```
1. 排重后值较少的列可使用`Bitmap`索引(`As 'BITMAP'`)；
2. 重建索引:
```sql
alter index employees_index
on table employees
partition (country='US')#指定重建某分区的索引,如果省略会重建所有分区的索引。
rebuild
;
```
3.显示索引:
```sql
show formatted index on employees
;
```
4.删除索引
```sql
drop index if exists employees_index
on table employees
;
```
如果原表被删除了，对其建立的对应的索引和索引表也会被删除。
如果原始表的某个分区被删除了，这个分区对应的分区索引也会被删除。

?:书上日期用的是数字'20110102'（`int`）,为何咱们用的是字符串'2015-11-02'?

>`NameNode`将所有系统元数据信息保存在内存中。一个`hdfs`实例所能管理的文件总数是有上限的，而`MapR`和`Amazon S3`则没有这个限制。

`hive`数据库和关系型数据库区别:
<table>
<tr>
 <td>关系型</td>
 <td>hive</td>
</tr>
<tr>
 <td>唯一键\主键\自增键</td>
 <td>无</td>
</tr>
<tr>
 <td>三范式(ACID)</td>
 <td>单行中存储一对多关系，一致性较差，但I/O性能高(连续存储)</td>
</tr>
<tr>
 <td>普通数据结构</td>
 <td>集合结构(array,map,struct)</td>
</tr>
</table>
`ACID`:原子性、一致性、隔离性、持久性。


- 一次扫描多次输出
```sql
from history
insert overwrite sales 
    select * where action='purchased'
insert overwrite credits
    select * where action='returned'
;
```

> ETL，Extraction-Transformation-Loading的缩写，
> 中文名称为数据提取、转换和加载。 

> ETL工具有：
> OWB(Oracle Warehouse Builder)、ODI(Oracle Data Integrator)、Informatic PowerCenter、Trinity、AICloudETL、DataStage、Repository Explorer、Beeload、Kettle、DataSpider

> ETL负责将分散的、异构数据源中的数据如关系数据、平面数据文件等抽取到临时中间层后进行清洗、转换、集成，最后加载到数据仓库或数据集市中，成为联机分析处理、数据挖掘的基础。

- 引用`hiveconf`变量:
```sql
$ hive -hiveconf dt=2011-01-01
insert overwrite table distinct_ip_in_logs
partition (hit_data=${dt})
select distinct(ip) as ip from weblogs
where hit_date='${hiveconf:dt}'
;
```

如何获得表中已有数据的规模信息(不用`count`)?
先用`show create table test1`查询到表在hdfs上的location,
然后用`dfs -du -h hdfs://location1`查询到文件大小。

- 分桶数据存储
1. 建表:
```sql
create table weblog (user_id INT,url String, source_ip String)
Partitioned by (dt string)
clustered by (user_id) into 96 buckets
;
```
使用`user_id`字段作为分桶字段，表数据按字段值哈希值分发到桶中。同一个`user_id`下的记录通常会存储到同一个桶内。假设用户数比桶数要多，那么桶内就将会包含多个用户的记录。

*结合`p109`的`map-side join`学习。*

2.分桶后插入数据:
```sql
set hive.enforce.bucketing=true;

From raw_logs
insert overwrite table weblog
partition (dt='2009-02-25')
select user_id,url,source_ip 
where dt='2009-02-25'
;
```

3.分桶表连接优化开启:
```sql
set hive.optimize.bucketmapJOIN=true;
#待连接的两个表分桶数量呈倍数关系时可优化。
# 且on语句中连接键与分桶键相同。
#sort-merge JOIN:
#分桶数完全相同时:
set hive.input.format=org.apache.hadoop.hive.ql.io.BucketizedHiveInputFormat;
set hive.optimize.bucketmapjoin=true;
set hive.optimize.bucketmapjoin.sortedmerge=true;
```

---

[SerDe][7] 全称是 `Serializer and Deserializer`
HDFS files –> InputFileFormat –> '<'key, value> –> Deserializer –> Row object
Row object –> Serializer –> '<'key, value> –> OutputFileFormat –> HDFS files

```flow
st=>start: HDFS files
e=>end: 结束 
op1=>operation: InputFileFormat
op2=>operation: '<'key,value>
op3=>operation: Deserializer 
op4=>operation: Row object
st->op1->op2->op3->op4->e 
```

```sql
explain select * from tmp_table1 limit 100;
OK #执行返回码
STAGE DEPENDENCIES:
  Stage-0 is a root stage

STAGE PLANS:
  Stage: Stage-0
    Fetch Operator
      limit: 100
      Processor Tree:
        TableScan
          alias: tmp_table1
          Statistics: Num rows: 0 Data size: 72 Basic stats: PARTIAL Column stats: NONE
          Select Operator
            expressions: id (type: string), perf (type: map<string,int>)
            outputColumnNames: _col0, _col1
            Statistics: Num rows: 0 Data size: 72 Basic stats: PARTIAL Column stats: NONE
            Limit
              Number of rows: 100
              Statistics: Num rows: 0 Data size: 72 Basic stats: PARTIAL Column stats: NONE
              ListSink

Time taken: 0.11 seconds, Fetched: 20 row(s)
```

```sql
hive> explain select count(*) from tmp_table1 limit 100;
OK
STAGE DEPENDENCIES:
  Stage-1 is a root stage
  Stage-0 depends on stages: Stage-1

STAGE PLANS:
  Stage: Stage-1
    Map Reduce
      Map Operator Tree:
          TableScan
            alias: tmp_table1
            Statistics: Num rows: 0 Data size: 72 Basic stats: PARTIAL Column stats: COMPLETE
            Select Operator
              Statistics: Num rows: 0 Data size: 72 Basic stats: PARTIAL Column stats: COMPLETE
              Group By Operator
                aggregations: count()
                mode: hash
                outputColumnNames: _col0
                Statistics: Num rows: 1 Data size: 8 Basic stats: COMPLETE Column stats: COMPLETE
                Reduce Output Operator
                  sort order:
                  Statistics: Num rows: 1 Data size: 8 Basic stats: COMPLETE Column stats: COMPLETE
                  TopN Hash Memory Usage: 0.3
                  value expressions: _col0 (type: bigint)
      Reduce Operator Tree:
        Group By Operator
          aggregations: count(VALUE._col0)
          mode: mergepartial
          outputColumnNames: _col0
          Statistics: Num rows: 1 Data size: 8 Basic stats: COMPLETE Column stats: COMPLETE
          Select Operator
            expressions: _col0 (type: bigint)
            outputColumnNames: _col0
            Statistics: Num rows: 1 Data size: 8 Basic stats: COMPLETE Column stats: COMPLETE
            Limit
              Number of rows: 100
              Statistics: Num rows: 1 Data size: 8 Basic stats: COMPLETE Column stats: COMPLETE
              File Output Operator
                compressed: false
                Statistics: Num rows: 1 Data size: 8 Basic stats: COMPLETE Column stats: COMPLETE
                table:
                    input format: org.apache.hadoop.mapred.TextInputFormat
                    output format: org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat
                    serde: org.apache.hadoop.hive.serde2.lazy.LazySimpleSerDe

  Stage: Stage-0
    Fetch Operator
      limit: 100
      Processor Tree:
        ListSink

Time taken: 0.055 seconds, Fetched: 50 row(s)
```

`explain extended`：
```sql
explain extended select count(*) from tmp_table1 limit 100;
OK
ABSTRACT SYNTAX TREE:

TOK_QUERY
   TOK_FROM
      TOK_TABREF
         TOK_TABNAME
            tmp_table1
   TOK_INSERT
      TOK_DESTINATION
         TOK_DIR
            TOK_TMP_FILE
      TOK_SELECT
         TOK_SELEXPR
            TOK_FUNCTIONSTAR
               count
      TOK_LIMIT
         100


STAGE DEPENDENCIES:
  Stage-1 is a root stage
  Stage-0 depends on stages: Stage-1

STAGE PLANS:
  Stage: Stage-1
    Map Reduce
      Map Operator Tree:
          TableScan
            alias: tmp_table1
            Statistics: Num rows: 0 Data size: 72 Basic stats: PARTIAL Column stats: COMPLETE
            GatherStats: false
            Select Operator
              Statistics: Num rows: 0 Data size: 72 Basic stats: PARTIAL Column stats: COMPLETE
              Group By Operator
                aggregations: count()
                mode: hash
                outputColumnNames: _col0
                Statistics: Num rows: 1 Data size: 8 Basic stats: COMPLETE Column stats: COMPLETE
                Reduce Output Operator
                  sort order:
                  Statistics: Num rows: 1 Data size: 8 Basic stats: COMPLETE Column stats: COMPLETE
                  tag: -1
                  TopN: 100
                  TopN Hash Memory Usage: 0.3
                  value expressions: _col0 (type: bigint)
                  auto parallelism: false
      Path -> Alias:
        hdfs://f04/user/hive/warehouse/temp.db/tmp_table1 [tmp_table1]
      Path -> Partition:
        hdfs://f04/user/hive/warehouse/temp.db/tmp_table1
          Partition
            base file name: tmp_table1
            input format: org.apache.hadoop.mapred.TextInputFormat
            output format: org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat
            properties:
              COLUMN_STATS_ACCURATE true
              bucket_count -1
              colelction.delim ,
              columns id,perf
              columns.comments
              columns.types string:map<string,int>
              field.delim
              file.inputformat org.apache.hadoop.mapred.TextInputFormat
              file.outputformat org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat
              location hdfs://f04/user/hive/warehouse/temp.db/tmp_table1
              mapkey.delim :
              name temp.tmp_table1
              numFiles 1
              serialization.ddl struct tmp_table1 { string id, map<string,i32> perf}
              serialization.format
              serialization.lib org.apache.hadoop.hive.serde2.lazy.LazySimpleSerDe
              totalSize 72
              transient_lastDdlTime 1438742089
            serde: org.apache.hadoop.hive.serde2.lazy.LazySimpleSerDe

              input format: org.apache.hadoop.mapred.TextInputFormat
              output format: org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat
              properties:
                COLUMN_STATS_ACCURATE true
                bucket_count -1
                colelction.delim ,
                columns id,perf
                columns.comments
                columns.types string:map<string,int>
                field.delim
                file.inputformat org.apache.hadoop.mapred.TextInputFormat
                file.outputformat org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat
                location hdfs://f04/user/hive/warehouse/temp.db/tmp_table1
                mapkey.delim :
                name temp.tmp_table1
                numFiles 1
                serialization.ddl struct tmp_table1 { string id, map<string,i32> perf}
                serialization.format
                serialization.lib org.apache.hadoop.hive.serde2.lazy.LazySimpleSerDe
                totalSize 72
                transient_lastDdlTime 1438742089
              serde: org.apache.hadoop.hive.serde2.lazy.LazySimpleSerDe
              name: temp.tmp_table1
            name: temp.tmp_table1
      Truncated Path -> Alias:
        /temp.db/tmp_table1 [tmp_table1]
      Needs Tagging: false
      Reduce Operator Tree:
        Group By Operator
          aggregations: count(VALUE._col0)
          mode: mergepartial
          outputColumnNames: _col0
          Statistics: Num rows: 1 Data size: 8 Basic stats: COMPLETE Column stats: COMPLETE
          Select Operator
            expressions: _col0 (type: bigint)
            outputColumnNames: _col0
            Statistics: Num rows: 1 Data size: 8 Basic stats: COMPLETE Column stats: COMPLETE
            Limit
              Number of rows: 100
              Statistics: Num rows: 1 Data size: 8 Basic stats: COMPLETE Column stats: COMPLETE
              File Output Operator
                compressed: false
                GlobalTableId: 0
                directory: hdfs://f04/home/maintain/hive/hive/tmp/hive-maintain/maintain/13951f1a-3bd6-481f-a901-1b2c185ec877/hive_2015-11-05_14-37-47_859_3395572406114851553-1/-ext-10001
                NumFilesPerFileSink: 1
                Statistics: Num rows: 1 Data size: 8 Basic stats: COMPLETE Column stats: COMPLETE
                Stats Publishing Key Prefix: hdfs://f04/home/maintain/hive/hive/tmp/hive-maintain/maintain/13951f1a-3bd6-481f-a901-1b2c185ec877/hive_2015-11-05_14-37-47_859_3395572406114851553-1/-ext-10001/
                table:
                    input format: org.apache.hadoop.mapred.TextInputFormat
                    output format: org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat
                    properties:
                      columns _col0
                      columns.types bigint
                      escape.delim \
                      hive.serialization.extend.nesting.levels true
                      serialization.format 1
                      serialization.lib org.apache.hadoop.hive.serde2.lazy.LazySimpleSerDe
                    serde: org.apache.hadoop.hive.serde2.lazy.LazySimpleSerDe
                TotalFiles: 1
                GatherStats: false
                MultiFileSpray: false

  Stage: Stage-0
    Fetch Operator
      limit: 100
      Processor Tree:
        ListSink

Time taken: 0.059 seconds, Fetched: 143 row(s)
```

```math
hive.exec.reducers.max=
(集群总reduce槽位个数*1.5)/(执行中的查询平均个数)
```

若小任务多可开启优化：推测执行和`JVM`重用。 

虚拟列?

I/O密集型该使用压缩；CPU密集型任务则不然。

`sequence file`存储格式: 压缩且可分。

`CLI`会话中通过`set`命令设置的属性在同一个会话中会一直生效的。

---

`show functions;`列出所有函数(包括用户自定义函数(`UDF`))。
`describe function extended concat;`展示函数的简单介绍。
`UDTF`自定义表生成函数；
`UDAF`自定义聚合函数。

---
`python`版`map-reduce`:
```python
# mapper.py:
import sys
for line in sys.stdin:
    words=line.strip().split()
    for word in words:
        print "%s\t" % (word.lower())
#reducer.py:
import sys

(last_key, last_count)=(None,0)
for line in sys.stdin:
    (key,count)=linde.strip().split("\t")
    if last_key and last_key!=key:
        print "%s\t%d" % (last_key, last_count)
        (last_key, last_count)=(key, int(count))
    else:
        last_key=key
        last_count+=int(count)
if last_key:
    print "%s\t%d" % (last_key, last_count)
```
使用`transform`关键字调用脚本，省得写`UDF`:
```sql
create table docs (line String)
;
create table word_count (word String, count Int)
Row format delimited fields teminated by '\t'
;
from(
    from docs
    select transform (line) using '${env:HOME}/mapper.py'
    as word,count
    cluster by word) wc
insert overwrite table word_count
select transform (wc.word,wc.count)
    using '${env:HOME}/reducer.py'
    as word, count
;
```

---

- 自定义序列化：
`json SerDe`:`P214`
```sql
create external table messages(
msg_id bigint,
tstamp string,
text string,
user_id bigint,
user_name string
)
row format SerDe "org.apache.hadoop.hive.contrib.serde2.JsonSerde"
with serdeProperties(
"msg_id"="$.id",
"tstamp"="$.created_at",
"text"="$.text",
"user_id"="$.user.id",
"user_name"="$.user.name"
)
Location '/data/messages'
;
```

---



```
hive -hiveconf start_date='2015-11-15' -hiveconf end_date='2015-11-21' -f ./comments_data.hql

solarWarehouse
userstat函数
```


这个地方错好多次了：
> left join 不在on里使用where
































































   
   
   
   
   
   
   
   


  [1]: http://git-scm.com/book/zh/v2
  [2]: http://pcottle.github.io/learnGitBranching/?demo
  [3]: http://blog.csdn.net/dc_726/article/details/41143175
  [4]: http://img.blog.csdn.net/20141115094556515?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvZGNfNzI2/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center
  [5]: http://img.blog.csdn.net/20141115094806194?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvZGNfNzI2/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center
  [6]: http://img.blog.csdn.net/20141115094934319?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvZGNfNzI2/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center
  [7]: http://www.coder4.com/archives/4031