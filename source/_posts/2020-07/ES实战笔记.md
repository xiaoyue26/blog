---
title: ES实战笔记
date: 2020-07-22 18:58:58
tags: 
- es
- elasticsearch
categories: es
---

# 索引结构
{% img /images/2020-07/lsm-tree.png 800 1200 lsm-tree %}
ES底层的lucene引擎的分段，比较类似LSM tree的机制。
分段不可变，合并生成新的大的分段。
ES中的translog对应Hbase中的WAL日志，防止进程崩了丢数据;


# 常用api
查看分析器对某段文本的结果:
```shell
curl -XPOST 'localhost:9200/get-together/_analyze?analyzer=myCustomAnalyzer' -d 'share your experience with NoSqlιbig data technologies'
```


组合内置分词器和过滤器,空格分词、小写+反转:
```shell
curl -XPOST 'localhost:9200/_analyze?tokenizer=whitespace&filters=lowercase,reverse' -d  'share your experience with NoSql big data technolog es'
```

查看单文档的所有token信息:(get-together索引下、group类型、文档id为1):
```shell
curl 'localhost:9200/get-together/group/1/_termvector?pretty=true'
```

# 分析器
分析器 = 0到1个字符过滤器 + 1个单个分词器 + 0到n个分词过滤器;

## 标准分析器(默认)
`standard analyzer` = 标准分词器 + 标准分词过滤器 + 小写转换分词过滤器 + 停用词分词过滤器
（0字符过滤+1分词器+3分词过滤器）

## 简单分析器
`simple analyzer`: 在非字母处进行分词 + 转小写

## 空白分析器
`whitespace analyzer`: 根据空白分词 + 0分词过滤器

## 停用词过滤器
`stop analyzer`: 根据停用词分词 + 0分词过滤器；

## 模式分析器
`pattern analyzer`: 允许指定一个分词切分模式;

## 语言分析器
包括汉语;

## 雪球分析器
`snowball analyzer`: 标准分析器 + 雪球词干器; 

# 分词器

## 标准分词器
主要处理欧洲语言,移除标点;

## 关键词分词器
整个文本提供给过滤器

## 字母分词器
基于非字母分词

## 小写分词器
非字母分词+转换成小写

## 空白分词器
通过空白来分词

## 模式分词器
例如可以在出现文本._.的地方分词:
```shell
curl -XPOST 'localhost:9200/pattern' -d '{
"settngs": {
    ”index” : {
        ”analysis”: {
            ”tokenizer": {
                ”patternl”: {
                    ”type": ”pattern”,
                    ”pattern”:”\\.-\\.” 
```

## UAX/URL电子邮件分词器
john.smith@example.com => 标准分词 
=>
john.smith
example.com

http://example.com?q=foo => 标准分词
http、example.com、q、foo

如果用UAX/URL电子邮件分词器，则可以保留:
john.smith@example.com(type:< EMAIL>)
http://example.com?q=bar(type:< URL>)

## 路径层次分词器
`path hierarchy tokenizer`
输入: /usr/local/var/log/es/log
分词结果: /usr、/usr/local、 .... /usr/local/var/log/es/log
因此有相同父目录的路径搜索(分词有相同部分)，能互相搜到。

# 分词过滤器
标准分词过滤器: 啥也不做;
小写过滤器、停用词过滤器、长度分词过滤器: 将最短和最长的单词过滤掉(自行设置min\max);
截断分词过滤器: 截断超出长度token;
修建分词过滤器: trim
限制分词数量分词过滤器: 限制最多多少个token被索引,比如设置max=8;
reverse分词过滤器: 把token反转，可以用于支持后缀索引;
唯一分词过滤器: 每个单词只保留第一次出现的位置(去重了)
ascii折叠分词过滤器: 尽量转ascii
同义词分词过滤器: 转成同义词
ngram过滤器: 略
滑动窗口分词过滤器: 略

# 提取词干
这个好像只是英文有用。把单词缩减到词根。
administrations -> administr
词干提取器: snowball,porter_stem,kstem
字典提取词干: hunspell分词过滤器+字典

# 打分相关
1. TF-IDF: 词频、逆文档频率
{% img /images/2020-07/tf-idf.png 800 1200 tf-idf %}
2. Okapi BM25;
3. 随机性分歧: DFR相似度
4. IB相似度;
5. LM dirichlet相似度;
6. LM Jelinek Mercer相似度;

## BM25
```json
{
"mappings":{
    "get-together": {
        "properties": {
            "title":{
                "type":"string"
                ,"similarity": "BM25"
            }
        }
    }
}
}
```

BM25的3个重要参数:
k1: 数值, 词频的重要性; (默认1.2)
b: 0~1数值, 篇幅对于得分的影响程度; (默认0.75)
discount_overlaps: 多个分词出现在同一位置，是否影响长度的标准化(默认true)


## boosting: 加权
可以用来修改文档相关性的程序。
包括:
- 索引期boosting
- 查询期boosting

一般使用查询期boosting(避免重新索引全部文档)



# 相关性、语义搜索的一些方案
首先所有的词向量模型都是基于分布假说的（distributional hypothesis）：拥有相似上下文的词，词义相似。

参考: https://zhuanlan.zhihu.com/p/80737146
1. word embedding
2. sentence embedding: 更难训练;

word embedding算法: 
word2vec：Skip-gram模型训练神经网络以预测句子中单词周围的上下文单词。
GloVe：单词的相似性取决于它们与其他上下文单词出现的频率。该算法训练单词共现计数的简单线性模型。
Fasttext：Facebook的词向量模型，其训练速度比word2vec的训练速度更快，效果又不丢失。

网上现有的预训练模型：基于维基百科语料库.

性能更优的方案:
1. 粗排: ES;
2. 精排: 语义模型计算相似度;

工业界主流: 谷歌的bert模型

# 中文分词IK相关
https://github.com/medcl/elasticsearch-analysis-ik

索引时优先使用`analyzer`配置的分词器，对文档进行分词;
// 索引时用ik_max_word,尽量多分几个词出来;
查询时优先使用`search_analyzer`配置的分词器，对输入进行分词;
// 查询时使用ik_smark, 尽量用最长的token去查询;
//
```shell
curl -XPOST http://localhost:9200/index/_mapping -H 'Content-Type:application/json' -d'
{
        "properties": {
            "content": {
                "type": "text",
                "analyzer": "ik_max_word",
                "search_analyzer": "ik_smart"
            }
        }

}'
```

# 实践遇到的问题

## 分词查询和关键字查询同时使用
分词查询的时候，切分是ik_max_word，最大只切到单词；
比如“工具”就是最小粒度了，因此如果查询的时候使用"工"则不会查询到结果。


如果是默认的标准分词器，则只会有单个字，不会有单词；

所以如果两个都要支持，可以用两个字段，（存两个字段）
一个字段用 ik_max_word， 一个字段用 standard;
查询的时候也是用bool or 连接，命中一个即可。



### 可参考的解决方案
用fields多加一个不分词的结果(name.raw)
```json
{
    "properties":{
        "name": {
                "type": "string",
                "analyzer": "standard", 
                "fields": {
                    "raw": {
                        "index": "not_analyzed",
                        "type": "string"
                    }
                    
                }
        
        }
    }
     
}
```


# 整合网上的近义词库
思路1: 自定义一个分词过滤器；// 可复用程度高
思路2: 写入该字段前，先用网上的近义词库把文本解析成空格分割的token，然后用空白分词器索引；（查询时用分词器） // 灵活，不用跟版本
// 由于可以配置多个分词过滤器，所以可以同时配置空格分词过滤器和同义词分词过滤器

维基百科近义词库： 528MB
http://licstar.net/archives/tag/wikipedia-extractor


某个领域最好的词向量:
http://licstar.net/archives/tag/%e8%af%8d%e5%90%91%e9%87%8f

考虑用boost加入相似度因素；（加权）

词向量资料：
http://licstar.net/archives/328

# 聚集
聚集有几个选项:
## 桶型聚集: (group by)
term: 词条聚集,就是统计文档数量;
significant_terms: 显著聚集
range: 范围聚集;
histogram: 直方图聚集;(类似范围，但是只需要提供间距即可)
嵌套聚集、反嵌套聚集、子聚集: 根据文档关系聚集;
地理距离聚集;


## 度量型聚集: (agg)
stats: 就是统计min,max,avg,count,sum信息;
extended_stats: 就是加上标准差这种更冷门的统计信息;
percentile: 分位数(近似,可以用compress参数控制精度和内存消耗)
cardinatily: 基数,也就是uv;// 近似的，hyperLogLog++, precision_threshold控制精度



## 过滤器和后过滤器
### 过滤器
{% img /images/2020-07/es-filter.png 800 1200 es-filter %}
### 后过滤器:
{% img /images/2020-07/es-post-filter.png 800 1200 es-post-filter %}

*两者区别*

文档->过滤器->查询->后过滤器->查询结果
文档->过滤器->查询->filter聚集->聚集结果

换句话说就是后过滤器不影响聚集，过滤器则影响聚集结果。
filter聚集则只影响聚集。

有一个例外是使用globel聚集，这样即使符合查询的只有2条文档，聚集也会应用到所有的文档上。(聚集比查询结果的数据源大)


# 文档间的关系
## 对象类型
输入:
```json
{
    "name": "name1"
    ,"events": [
        {"title": "hadoop"
        ,"date”: "12月"
        }
        ,{"title": "es"
        ,"date”: "6月"
        }
    ]
}
```
这种数据实际索引的时候，会把各个字段分别组成数组:
```json
events.title: ["hadoop","es"]
events.date: ["6月",“12月”]
```
所以搜的时候如果想搜6月的hadoop, 也可以搜出12月hadoop的文档(name1).

## 嵌套类型
上面的情况可以用嵌套类型解决。
这个时候的索引:
```json
[events.title: hadoop
events.date: 12月
,
events.title: es
events.date: 6月

]

```

## 父子关系和反规范化
父子关系的存储：
1.规范化：父文档和子文档分开存储，然后再存储一个映射关系；// 相关查询: has_parent/has_child
2.反规范化：子文档中存储父文档；（空间换时间）

## 嵌套json的存储
由于ES的底层Lucence只支持扁平结构，ES支持嵌套json的方法是通过强行打平,如：
```json
{
"title": "titl1"
,"location":{
    "name": "name1"
    ,"geolocation": "51.52,-0.09"
}
}
```
实际实施到Lucence层的时候是这样存的:
```yaml
titile: "title1"
location.name: "name1"
location.geolocation: "51.52,-0.09"
```
因此我们设计的key一定不要有小数点符号。
而且最好是一对一关系（不是数组）。

父子关系的索引选项：
include_in_parent/include_in_root

### 一对一
嵌套json/对象

#### 一对多
嵌套文档: 索引阶段进行join; // 同分片存储，保证本地连接
父子关系: 查询阶段进行join; // 不同分片,远程连接

#### 多对多
反规法化: 可以处理多对多关系

# ES扩展
ES集群使用master-slaver架构，master和slaver用心跳信息来判断彼此的存活；
（有点类似hadoop，不知道是不是也有hadoop的HA；hadoop在120个节点的时候namenode容易OOM，不知道ES有没有类似问题）
master\slaver互相ping应该会消耗一些带宽，可以考虑调节心跳频率调节性能。

节点下线：先停用（停止数据写入、迁移）

## 集群升级
### 重启
直接关闭整个集群，不可用。
然后升级所有节点，重启集群。
有一段时间不可用。

### 轮流重启
不牺牲可用性的情况下，重启集群；
基本步骤是：关一个节点，升级一个节点，重启这个节点，重新加入集群。
这里有一个关键就是，关闭某个节点的期间，不需要集群自己做rebalance.
因此配置：`cluster.routing.allocation.enable`=`none`
可以用curl发命令修改这个配置。
过后重新设置为`all`。

如果副本数>1，上述操作期间服务依然可用。

## 别名API
```json
curl -XPOST 'localhost:9200/_aliases' -d '
{
    "actions": [
        {
            "add": {
                "index": "get-together",
                "alias": "gt-alias"
            }
        },
        {
            "remove": {
                "index": "old-get-together",
                "alias": "gt-alias"
            }
        }
    ]
}'
```
可以分拆成两个命令：
```shell
curl -XPUT 'http://localhost:9200/get-together/_alias/gt-alias'
curl -XDELETE 'http://localhost:9200/old-get-together/_alias/gt-alias'
```
一个别名可以指向多个索引，甚至指向logs-开头的索引。
(类似于一个逻辑名称)
别名还可以附带一个过滤器。


## 路由
默认路由策略：文档id
可以手动指定routing=xxx来影响分片行为。

因此可以根据业务，把一起访问的文档路由到同一分片上。
### debug api: 查看搜索的分片
routing为xxx时，会搜索哪个分片:
```shell
curl -XGET 'localhost:9200/get-together/_search_shards&routing=xxxx'
```


可以在别名中配置路由，简化查询操作。

# 性能优化
## 合并请求(bulk接口)
批量新增、批量更新、批量搜索

## IO配置优化
segments: 分段;

ES接收到文档后:
分段的倒排索引

### 概念
refresh: 刷新; 生效，重新打开索引;新建的索引生效, 以前的缓存失效;
flush: 冲刷; 刷盘，索引数据写入磁盘; 
合并: 小分段合并成大分段; // 分段越多，查询越慢;
存储限流: 调节每秒写入的字节数;

### 优化策略
#### 刷新频率降低
缓存失效的频率降低，性能更高，新文档慢一些生效；
// 默认每秒刷新, index.refresh_interval

#### 刷盘频率降低
IO消耗降低，性能更高，丢数据概率提高; 
触发刷盘的时机:
1.内存缓存区已满;
2.固定间隔(定时器);
3.事务日志达到阈值;
因此调控的手段:
1.内存缓存区大小: indices.memory.index_buffer_size;
2.刷新间隔: index.translog.flush_threshold_period;
3.事务日志大小: index.translog.flush_threshold_size;

### 合并策略优化
合并的作用：
1. 真正删除文档;
2. 分段越少，查询越快;

触发合并的时机:
1. 索引文档;
2. 更新、删除文档;

合并相关配置:
`index.merge.policy.segments_per_tier`:
每层的分段数量;
高=>写性能越好，越低=>读性能越好;
`index.merge.policy.max_merge_at_once`:
每次合并多少分段; 设置为等于`segments_per_tier`即可;
`index.merge.policy.max_merged_segment`:
分段的最大规模;
低=>写性能好; 高=>读性能好;
`index.merge.scheduler.max_thread_count`:
合并用的最大线程数;

### 存储限流
`indices.store.throttle.max_bytes_per_sec`: 
最大IO吞吐量，默认20MB/s，默认只针对merge(合并分段);
(ssd的话，可以调大到100～200MB)

`indices.store.throttle.type`:
限流类型；`none`: 不限流, `all`: 所有磁盘操作; 默认: `merge`。


### 磁盘IO优化
MMAPDirectory:
进程请求OS对磁盘文件进行内存映射(初始化开销);
也就是mmap，0拷贝，进程挂掉的话，内核会帮忙保存文件;

NIOFDirectory:
进程将磁盘文件复制到JVM堆中;
也就是常规文件访问，进程挂掉，则文件修改丢失;

相关配置:
`index.store.type`: 默认`default`.
mmapfs: 只使用MMapDirectory, 静态索引，物理内存能放下索引时适用;
niofs: 只使用NIOFSDirectory,32位系统适用;

可以对单个索引配置，也可以配置成全局。

## 缓存优化
ES的缓存分为:
1. 分片查询缓存: 缓存查询结果;
2. OS缓存: 缓存索引到内存;

## 过滤器缓存
过滤器缓存可以在query时，在filter用`_cache`:true/false配置。
过滤器缓存在各个节点上，内存占比配置:
`indices.cache.filter.size`: 默认10%
缓存淘汰策略: LRU
缓存生存时间: 
`index.cache.filter.expire`: 30m  (表示30分钟过期)

比较简单的过滤器可以使用bitset来减少内存消耗；
比较复杂的过滤器则直接存储查询结果。
可以使用bitset的过滤器:
term,exists/missing,prefix

## 字段过滤器
索引：token -> 文档 ; (又叫倒排索引)
字段: 文档 -> 词条; (主要用于排序和聚集)

字段上可以用的过滤器:
terms过滤器
range过滤器

## 预热器优化
可以在索引上定义预热器。
即将按日期倒序查询:
```shell
curl -XPUT 'localhost:9200/get-together/event/_warmer/upcoming_events' -d '{
    "sort":[
        {
            "date": {"order":"desc"}
        }
    ]

}'
```
即将查询热门分组:
```shell
curl -XPUT 'localhost:9200/get-together/group/_warmer/top_tags' -d '{
    "aggs":{
        "top_tags":{
            "terms":{
                "field": "tags.verbatim"
            }
        }
    }
}'
```
可以创建索引的时候直接定义预热器。

## 脚本优化
如果只有数值型的操作，可以考虑用lucene表达式代替脚本；
性能：
不用脚本>lucene表达式（js）>java脚本>其他脚本

## ES查询过程
### topN查询
比如取10个结果:
1.每个分片取得分前10的;
2.合并所有分片的结果(归并排序);

因此得分是在每个分片上算的（分片内得分）

### 分页查询
{
"from": 400
,"size": 100
}
这种需要400～500的结果，但实际会取前500个，然后扔掉前400个。

如果是顺序翻页需求，可以用scroll类型查询优化，查询的时候传递一个scroll=1m的参数，让ES等一分钟，以准备接收下一次翻页。这种情况下，每次ES都会返回一个scrollId。

# 集群管理
## 模版
可以配置某个前缀的索引都应用某个配置模版。
（比如模版里定义别名）

### 多个模版的合并
多个模版可能匹配到同一个索引，这个时候多个模版的配置会合并。
合并的顺序按照order字段，0的先执行，然后1的覆盖0的，依此类推。







