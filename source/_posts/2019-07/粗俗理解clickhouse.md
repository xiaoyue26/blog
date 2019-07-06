---
title: 粗俗理解clickhouse
date: 2019-07-06 17:06:37
tags: clickhouse
categories: clickhouse

---


# what: clickhouse是啥?
clickhouse是俄罗斯开源的一个用于OLAP分析的核心引擎，它可以基于海量的日志数据接受类sql查询，以秒~分钟量级的延迟返回查询结果。
它目前应用在了俄罗斯的搜索引擎Yandex.Metrica中、欧洲核子研究中心: PB级存储、统计分析查询，以及我国各大互联网公司的BI后台引擎中。

## 应用： Yandex.Metrica
2014年: 每天120亿个事件。（点击、浏览）
374台服务器，20.3万亿行数据。
压缩后: 2PB 
压缩前: 17PB

详细介绍官网:
https://clickhouse.yandex/docs/zh/
开源代码:
https://github.com/yandex/ClickHouse
中文文档:
https://github.com/yandex/ClickHouse/tree/master/docs/zh

# why: 为啥选择clickhouse?
主要原因有: 性能高、跑分高、功能多、可用性高。

## 性能高、跑分高
// 俄罗斯的程序员在算法方面的活跃度排名世界第一
// C++实现、老毛子轻易不开源,参见nginx

> 摘自知乎: https://zhuanlan.zhihu.com/p/22165241
1亿数据:
比Vertica快5倍,比Hive快279倍,比Mysql快801倍;
10亿数据:
比Vertica快5倍,mysql无法完成。

### 单机性能
有page cache: 2-10GB／s（未压缩），上限30GB/s
无page cache: 1.2G/s(压缩率3)
(磁盘400MB/s)如果是10B的列，就是1-2亿行/s。 

## 功能多
最重要的是有`AggregatingMergeTree`表引擎，专门优化了三个数据分析最实用的查询:（海量数据快速计算）
```
uniq: 计算uv
any: 抽样统计
quantiles: 分位数
```
上述几个功能如果用sparkSql/hive,一般耗时都是15分钟以上。(甚至到半小时、1小时)
如果用mysql的话,则由于维度爆炸的问题可能存不下这么多数据，并且无法灵活新增维度。
clickhouse对于海量数据处理没有spark/hive那么灵活,但是特化了OLAP的即时查询性能,本质上是处在不同领域的工具。
从数据仓库的角度来看:
ods层: 用spark/hive进行ETL后产生;
dw层: ods载入clickhouse后直接产生预聚合的数仓,支持即时查询;
dm层: mysql

### 对比
hbase/ES:一般用来支持海量数据点查询;
mysql: 用来支持无聚合的点查询;
clickhouse: 用来支持海量数据的聚合查询。
kylin: 比较接近clickhouse，底层是hbase+星型模型

其他引擎:
#### ReplcingMergeTree
删除相同主键的重复项(去重)
### SummingMergeTree
将一个part中，相同主键的所有行聚合成一行，包含一系列聚合函数状态。
### CollapsingMergeTree
提供折叠行功能： 把同主键的数据行去重到最多两行。（再次强调所有聚合都在part内）
场景: 用户访问状态记录、频繁变化的数据

前面说的clickhouse不支持update数据，所以用这个引擎可以近似达到一部分update的效果。
本质上就是类似于git的revert、银行系统里的冲正、mysql的MVCC。

比如我们要记录用户访问情况，先插入一条:
`userid_0,5,146,1` 表示0号用户访问了5个页面，停留146秒(最后一列的1暂时忽略)。
然后过了一会儿想改成它访问了6个页面，停留185秒，那就插入:
```
userid_0,5,146,-1
userid_0,6,185,1
```
首先把原来的取消掉，标记列-1。然后插入最新的状态数据，标记列1.

> 应当注意这些成对的1,-1会异步地被删除，所以不能查到状态变化历史,仅用于查最新。

这种引擎的建表语句:
```sql
CREATE TABLE UAct
(
    UserID UInt64,
    PageViews UInt8,
    Duration UInt8,
    Sign Int8
)
ENGINE = CollapsingMergeTree(Sign)
ORDER BY UserID
```
查询的时候的语法:
```sql
SELECT
    UserID,
    sum(PageViews * Sign) AS PageViews,
    sum(Duration * Sign) AS Duration
FROM UAct
GROUP BY UserID
HAVING sum(Sign) > 0
```

`CollapsingMergeTree`要求插入的顺序不能乱来，要按状态的变化顺序。
如果顺序无法保证，可以使用`VersionedCollapsingMergeTree`,它的算法也很简单,就是要求用户多传一个version字段。
### GraphiteMergeTree
直接接到日志收集。
可以存metrics指标可视化系统Graphite的rollup数据。
如果不rollup，可以用别的引擎。


### Log系列的引擎(非主打)
用于小数据量(< 100w)的表。
包括: StripeLog,Log,TinyLog三个引擎。

特性:
- 追加写,不支持改
- 不支持索引
- 非原子写入(可能有损坏的数据)

TinyLog：最简单的表引擎，适合一次写入即终身、多次查询的小数据，不支持并发数据访问，不支持同时写入同时读取。
Log：比TinyLog多一个偏移量优化.
Memory：以直接形式存储在内存中，读写变态快，但是记住是临时的，关机数据消失。
Buffer：缓冲，可以理解为将数据存储在内存中，然后达到一定阈值限制条件，那么先前的数据会自动写入设定的表格中。这样可以将部分热数据放在内存中缓存，快速访问和读取，而时间较为久远的数据写入表中释放内存，应该比较好理解。（可以实时盯数据）
External data：从字面理解，就是可以将文件数据等引入query语句中利用了。比如你想查找一些在你所给的名单中的用户的消费数据，那么你可以免除复制粘贴，直接将这个名单文件引入并使用，clickhouse会自动给这个文件建立一个临时表。

### 其他功能:
- 支持类SQL查询，相应的库函数很多：ip转换、数组、map、url转换、近似
计算uv、近似计算分位数、抽样统计等等；
https://clickhouse.yandex/docs/zh/query_language/select/
- 数据源支持繁多，可以是以kafka、tcp、jdbc、文件等等直接作为表。
- webUI支持: https://tabix.io/
- IDE支持: jetbrain打造的datagrip插件: https://blog.jetbrains.com/datagrip/tag/clickhouse/
http://www.clickhouse.com.cn/topic/5b6ce6359d28dfde2ddc6229

## 可用性高
任何时候随时可以给表添加字段、属性、维度，不会拖慢或影响集群运行速度。
BI系统很大的一个痛点是维度的**组合爆炸**，而且经常需要新增，clickhouse针对性地优化了这一点。(如果是mysql要新增维度列,需要重做整个表,即使是mysql8的瞬加字段也不行)

流水线式的数据处理流程，数据一旦进入系统，那么立即处于可以使用的状态，边读（查询）边写没有任何压力。


# How: clickhouse的底层实现原理
主要思想是根据OLAP的特征舍弃了一部分功能，然后针对性地优化了一部分功能。主要方法包括LSM（MergeTree系列表引擎）、稀疏索引（缓存友好）、列式存储+数据压缩、VectorWise、用概率算法进行近似等等。

## 需求分析
### OLAP应用的特点:
1. 大多数是读请求
2. 数据总是以相当大的批(> 1000 rows)进行写入
3. 不修改已添加的数据
4. 每次查询都从数据库中读取大量的行，但是同时又仅需要少量的列
5. 宽表，即每个表包含着大量的列
6. 较少的查询(通常每台服务器每秒数百个查询或更少)
7. 对于简单查询，允许延迟大约50毫秒
8. 列中的数据相对较小： 数字和短字符串(例如，每个URL 60个字节)
9. 处理单个查询时需要高吞吐量（每个服务器每秒高达数十亿行）
10. 事务不是必须的
11. 对数据一致性要求低
12. 每一个查询除了一个大表外都很小
13. 查询结果明显小于源数据，换句话说，数据被过滤或聚合后能够被盛放在单台服务器的内存中

### 面临的困难：
1. 维度组合爆炸；
2. 聚合数据后,如果有修改很蛋疼.
3. URL这种无法预聚合.

### 需求洞察:
用户只关心聚合后中极小一部分

市场上的备胎: sparkSQL,Impala,Drill都不好用。

### 舍弃的功能
1. 事务支持
2. 快速修改、删除数据。 (可以低速批量删除、修改)
3. 点查询(检索单行): 因为用的是稀疏索引。
(好处是稀疏所以索引能完全放入内存，范围查询很快)
4. 高并发查询: 只支持100/s量级查询,对于内网应用、分析型业务足够。 
5. 窗口函数。

## 实现
基于上述几点需求分析的优化:
1. cpu: VectorWise方法,将压缩的列数据整理成现代CPU容易处理的Vector模式。利用现代CPU的多线程。 SIMD: 每次处理一批Vector数据。
2. 提高内存利用率: 稀疏索引;
3. 硬盘: MergeTree系列表引擎(LSM算法),批量合并写入,提高IO吞吐率;
4. 算法: 近似算法/概率算法。

架构: 表=>shard=>replica=>partiton=>part

## 稀疏索引

对应`index_granularity`参数:
```c
M(SettingUInt64, index_granularity, 8192, "How many rows correspond to one primary key value.") \
```
索引中相邻mark之间的数据行数,默认8192.
借助稀疏索引，它能存更多的索引在内存中。（相当于存了B树的前几层或二级索引）。


其他配置:
https://github.com/yandex/ClickHouse/blob/master/dbms/src/Storages/MergeTree/MergeTreeSettings.h

比如io配置:
```c
    M(SettingUInt64, min_merge_bytes_to_use_direct_io, 10ULL * 1024 * 1024 * 1024, "Minimal amount of bytes to enable O_DIRECT in merge (0 - disabled).") \
```
超过多少Bytes以后绕过内核缓冲，进行直接IO。(节省内存开销、数据复制开销)

其他配置的分三大块:
```c
    /** Merge settings. */ \    合并时的配置
    /** Inserts settings. */ \  插入时的配置
    /** Replication settings. */ \ 副本的配置
    /** Check delay of replicas settings. */ \ 副本检查延迟配置
    /** Compatibility settings */ \   兼容性配置
```

### 稀疏索引示例
数据存储:
```
全部数据  :     [-------------------------------------------------------------------------]
CounterID:      [aaaaaaaaaaaaaaaaaabbbbcdeeeeeeeeeeeeefgggggggghhhhhhhhhiiiiiiiiikllllllll]
Date:           [1111111222222233331233211111222222333211111112122222223111112223311122333]
标记:            |      |      |      |      |      |      |      |      |      |      |
                a,1    a,2    a,3    b,3    e,2    e,3    g,1    h,2    i,1    i,3    l,3
标记号:          0      1      2      3      4      5      6      7      8      9      10
```
1. CounterID in ('a', 'h'): [0, 3) 和 [6, 8) 区间
2. CounterID IN ('a', 'h') AND Date = 3 : [1, 3) 和 [7, 8) 区间
3. Date = 3: 扫全表。

表由按主键排序的数据 `part` 组成。
当数据被插入到表中时，会分成`part`并按主键的字典序排序。例如，主键是 (CounterID, Date) 时，part中数据按 CounterID 排序，具有相同 CounterID 的部分按 Date 排序。

不会合并来自不同分区的数据片段。（性能考虑）
不保证相同主键的所有行都会合并到同一个数据片段中。(没有必要)

索引文件： 每个part创建一个
每隔index_granularity一个索引行号(mark)；
对于每列，跟主键相同的索引行处也会写入mark。这些mark让你可以直接找到数据所在的列。

## 表引擎：MergeTree族引擎
表引擎（即表的类型）决定了：
> 数据的存储方式和位置，写到哪里以及从哪里读取数据
支持哪些查询以及如何支持。
并发数据访问。
索引的使用（如果存在）。
是否可以执行多线程请求。
数据复制参数。

clickhouse中最强大的都是合并树引擎系列。

- 理念:
批量写入,后台合并;

- 特点:
1. 数据按主键排序; (类似于聚簇)
2. 允许使用主键分区; (类似于Hive)
3. ReplicatedMergeTree系列支持副本(类似于hdfs)
4. 支持数据采样;(类似于Mysql performanceSchema)

建表语句:
```sql
CREATE TABLE [IF NOT EXISTS] [db.]table_name [ON CLUSTER cluster]
(
    name1 [type1] [DEFAULT|MATERIALIZED|ALIAS expr1],
    name2 [type2] [DEFAULT|MATERIALIZED|ALIAS expr2],
    ...
    INDEX index_name1 expr1 TYPE type1(...) GRANULARITY value1,
    INDEX index_name2 expr2 TYPE type2(...) GRANULARITY value2
) ENGINE = MergeTree()
[PARTITION BY expr]
[ORDER BY expr]
[PRIMARY KEY expr]
[SAMPLE BY expr]
[SETTINGS name=value, ...]
```
示例语句:
```sql
ENGINE MergeTree() 
PARTITION BY toYYYYMM(EventDate) 
ORDER BY (CounterID, EventDate, intHash32(UserID)) 
SAMPLE BY intHash32(UserID) 
SETTINGS index_granularity=8192
```
默认情况下主键跟排序键（由 ORDER BY 子句指定）相同。
这里可以看出它不支持唯一索引,重复是很自然的。由上层自己保证。


SummingMergeTree 和 AggregatingMergeTree 引擎中
列分为两种:

- 维度
- 度量 (各种pv,uv等等)


Mysql的做法是把所有维度作为主键; (每次新增维度很痛)
clickhouse的推荐做法是把旧的维度作为主键(保留少量),所有维度(旧维度+新维度)作为排序列。
这里排序列的修改是轻量级的:
旧的维度是整体排序列的前缀(已然有序)，仅需排序新加的行。

推荐使用方案:
原始数据=> MergeTree （确保原始数据不丢失）
原始数据=> SummingMergeTree/AggregatingMergeTree (得到预聚合数据)

> 引擎会定期合并相同主键的数据进行聚合。最终结果中多半还是有重复主键，但是同一个part中不会有。

具体来说：
`SummingMergeTree`: 把相同排序列的行聚合。
被聚合的列在建表语句中通过`columns`指定。（数值、非主键）
(如果`columns`为空会聚合所有非排序列)

特殊情况:
1. 某行所有度量列值都是0，直接删除该行;(sum优化)
2. 非数值(无法汇总): 随机选一个值.
3. 支持sumMap函数: 某列是map结构。


#### AggregatingMergeTree引擎
`SummingMergeTree`只支持算pv,`AggregatingMergeTree`能支持算uv,分位数,抽样,三个函数:
```
uniq
anyIf (any+If)
quantiles
```

**创建:**(物化视图)
```sql
CREATE MATERIALIZED VIEW test.basic
ENGINE = AggregatingMergeTree() PARTITION BY toYYYYMM(StartDate) 
ORDER BY (CounterID, StartDate)
AS SELECT
    CounterID,
    StartDate,
    sumState(Sign)    AS Visits, -- 聚合1: pv
    uniqState(UserID) AS Users  -- 聚合2: uv 注意是记录了状态(特定的二进制表示法)
FROM test.visits
GROUP BY CounterID, StartDate;
```
插入数据的时候只需要插入到`test.visits`.
视图中也会有数据，并且会聚合。


**查询:**
```sql
SELECT
    StartDate,
    sumMerge(Visits) AS Visits, -- 注意都变成了merge后缀
    uniqMerge(Users) AS Users
FROM test.basic
GROUP BY StartDate
ORDER BY StartDate;
```

## 算法: uniq
上一节中`AggregatingMergeTree`的uniq求uv,其实有三个函数:
```
uniq: 用UniquesHashSet近似求uv（BJKST算法）
uniqHLL12: 用HLL近似求uv 
uniqExact: 用HashSet精确求uv
```

源码见:
https://github.com/yandex/ClickHouse/blob/master/dbms/src/AggregateFunctions/AggregateFunctionUniq.cpp

其中HLL就是HyperLogLog算法。

而第一个`UniquesHashSet`(https://github.com/yandex/ClickHouse/blob/ef50601b5ceeeaf5763eab6c0013954c12eb00b1/dbms/src/AggregateFunctions/UniquesHashSet.h)
两者的思想都是uv越大,不同的hash值越多。



`UniquesHashSet`的特点是内存消耗小,性能高。
具体实现是将输入hash到UInt32,然后插入到数组中,如果遇到碰撞则进行线性探测. (原始输入丢弃,只存hash值)随着插入进行达到阈值`UNIQUES_HASH_MAX_SIZE`时，则将当前存的值丢弃一半,只保留能整除2的值,提高`skip_degree`值,然后开始只接受能整除2的输入。依此类推,后续就是只接受整除4，8，16的值。最后获取结果:
```c
 size_t size() const
    {
        if (0 == skip_degree)
            return m_size;
        size_t res = m_size * (1ULL << skip_degree);
        /** Pseudo-random remainder - in order to be not visible,
          * that the number is divided by the power of two.
          */
        res += (intHashCRC32(m_size) & ((1ULL << skip_degree) - 1));
        /** Correction of a systematic error due to collisions during hashing in UInt32.
          * `fixed_res(res)` formula
          * - with how many different elements of fixed_res,
          *   when randomly scattered across 2^32 buckets,
          *   filled buckets with average of res is obtained.
          */
        size_t p32 = 1ULL << 32;
        size_t fixed_res = round(p32 * (log(p32) - log(p32 - res)));
        return fixed_res;
    }
```

rehash的实现:
```c
void rehash()
    {
        for (size_t i = 0; i < buf_size(); ++i)
        {
            if (buf[i] && !good(buf[i]))
            {
                buf[i] = 0;
                --m_size;
            }
        }

        /** After removing the elements, there may have been room for items,
          * which were placed further than necessary, due to a collision.
          * You need to move them.
          */
        for (size_t i = 0; i < buf_size(); ++i)
        {
            if (unlikely(buf[i] && i != place(buf[i])))
            {
                HashValue x = buf[i];
                buf[i] = 0;
                reinsertImpl(x);
            }
        }
    }
```
其中good函数含义就是能否被2^skip_degree整除。

- 线性探测：
为了加快速度，增加了一个假设: 所有数据只插入Key/更新Key，不删除Key。
(这个假设在大数据处理/统计的场景下，大多都是成立的，spark中openHashSet也是线性探测)
有了这个假设它可以去掉拉链表，使用线性探测来实现哈希表。
- 内存利用率高: 去掉了8B指针结构，能够创建更大的哈希表，冲突减少；
- 内存紧凑: 位图操作快，一个内存page就能放下很多位图，8B就能放64个位置，缓存友好(while循环pos++)。



## 存储
假如表结构是:
```sql
create table test.mergetree1 
(sdt  Date
, id UInt16
, name String
, cnt UInt16) 
ENGINE=MergeTree(sdt, (id, name), 10);
```
分区字段是日期sdt.
对应的目录结构:
```sql
├── 20180601_20180601_1_1_0
│   ├── checksums.txt
│   ├── columns.txt -- 元数据
│   ├── id.bin -- 压缩列
│   ├── id.mrk -- 索引mark
│   ├── name.bin
│   ├── name.mrk
│   ├── cnt.bin
│   ├── cnt.mrk 
│   ├── cnt.idx
│   ├── primary.idx -- 主键
│   ├── sdt.bin
│   └── sdt.mrk -- 保存一下块偏移量
├── 20180602_20180602_2_2_0
│   └── ...
├── 20180603_20180603_3_3_0
│   └── ...
├── format_version.txt
└── detached -- 破损数据
```

# 总结
clickhouse为啥比hive/spark快:
- 7*24小时都在后台预聚合.hive/spark计算的时候才申请资源,平时只占一点点;
- 可以用星型模型缩减数据类型、压缩友好;
- 计算过程没有hive/spark中的shuffle概念,全是mapAgg;

clickhouse为啥比mysql快:(仅限clickhouse擅长的查询)
- 预聚合
- 多核优化、vector优化更彻底
- 分区+稀疏索引,整个索引能放内存,然后并发查part(这点还是要结合多核优化)
- 根据排序键排序存放 

优化的方面:
1. cpu: VectorWise方法,将压缩的列数据整理成现代CPU容易处理的Vector模式。利用现代CPU的多线程。 SIMD: 每次处理一批Vector数据。
2. 提高内存利用率: 稀疏索引;
3. 硬盘: MergeTree系列表引擎(LSM算法),批量合并写入,提高IO吞吐率,牺牲随机读能力;
4. 算法: 近似算法/概率算法,HLL\BJKST算法等。

