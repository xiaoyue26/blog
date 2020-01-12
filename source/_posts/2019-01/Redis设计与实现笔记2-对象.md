---
title: Redis设计与实现笔记2-对象
date: 2019-01-12 19:58:57
tags:
- redis
categories:
- redis

---

参考\搬运资料: http://redisbook.com/index.html
 

# 对象
上一节的数据结构都是底层数据结构的实现，也就是`object encoding xxx`的结果。（redis称之为`编码`）
封装一下的话，逻辑上看做一个对象：
```c
#define LRU_BITS 24
typedef struct redisObject {    // redis对象
    unsigned type:4;    // 类型,4bit
    unsigned encoding:4;    // 编码,4bit
    unsigned lru:LRU_BITS; /* lru time (relative to server.lruclock) */ // 24bit
    int refcount;   // 引用计数
    void *ptr;  // 指向各种基础类型的指针
} robj;
```
`lru`: 上次访问的时间(从服务器最初时钟时间开始);(秒)
要计算对象多久没访问了:
空转时长=当前时间-lru时间。
查看对象空闲时间:(秒)
```
OBJECT IDLETIME msg
```
2^24s=>大概194天。
redis中lru的具体实现逻辑:
https://zhuanlan.zhihu.com/p/34133067

成本较高,redis4后引入lfu,效率近似于原来的lru.(因为原来就是偏概率的算法,不是标准的lru)


类型:

| 类型常量      | 对象的名称   |   
| :--------:   | :----- |  
| REDIS_STRING | 字符串对象  | 
| REDIS_LIST | 列表对象|
| REDIS_HASH | 哈希对象|
| REDIS_SET | 集合对象|
| REDIS_ZSET | 有序集合对象|

五种基本对象（`type`,类型）：字符串、列表、哈希、集合、有序集合

获取类型:
```redis
SET msg "hello world"
TYPE msg # 返回string
```


编码`encoding`:

| 编码常量        | 编码所对应的底层数据结构   |   
| :--------:   | :----- |  
| REDIS_ENCODING_INT | long 类型的整数  | 
| REDIS_ENCODING_EMBSTR | embstr编码的简单动态字符串|
| REDIS_ENCODING_RAW | 简单动态字符串|
| REDIS_ENCODING_HT | 字典|
| REDIS_ENCODING_LINKEDLIST | 双端链表|
| REDIS_ENCODING_ZIPLIST | 压缩列表|
| REDIS_ENCODING_INTSET | 整数集合|
| REDIS_ENCODING_SKIPLIST | 跳跃表和字典|

5种类型可能的8种编码。


类型和编码的可能对应:

| 类型        | 编码   |   对象 |
| :--------:   | :----- | :--------:   
|REDIS_STRING | REDIS_ENCODING_INT | 使用整数值实现的字符串对象| 
|REDIS_STRING | REDIS_ENCODING_EMBSTR | embstr编码的简单动态字符串|
|REDIS_STRING | REDIS_ENCODING_RAW | 简单动态字符串|
|REDIS_LIST   | REDIS_ENCODING_ZIPLIST | 使用压缩列表实现的列表对象|
|REDIS_LIST   | REDIS_ENCODING_LINKEDLIST | 使用双端链表实现的列表对象|
|REDIS_HASH | REDIS_ENCODING_ZIPLIST | 使用压缩列表实现的哈希对象|
|REDIS_HASH | REDIS_ENCODING_HT | 使用字典实现的哈希对象|
|REDIS_SET |REDIS_ENCODING_INTSET | 使用整数集合实现的集合对象|
|REDIS_SET |  REDIS_ENCODING_HT |   使用字典实现的集合对象 |
|REDIS_ZSET | REDIS_ENCODING_ZIPLIST|  使用压缩列表实现的有序集合对象 |
|REDIS_ZSET | REDIS_ENCODING_SKIPLIST|  使用跳跃表和字典实现的有序集合对象|
 

# 类型与编码
类型5种：
字符串、列表、哈希表，集合、有序集合

每种类型底层可以有不同实现，也就是不同的编码。

# 对象之——字符串
三种实现: `int`、`embstr` 、`raw`

- `int`: long能存下的数会用数字编码;
- `embstr`: <=39B的字符串，或者浮点苏，会用`embstr`编码;
- `raw`: >39B的字符串，会用`raw`编码,也就是简单动态字符串(`SDS`)。

{% img /images/2019-01/raw.png 800 1200 raw %}
{% img /images/2019-01/embstr.png 800 1200 embstr %}
`embstr`顾名思义就是嵌套字符串，把元数据和实际数据存放在一起，好处是这样可以对缓存友好，可以减少一次寻址。坏处是只能支持短字符串。

上述3种字符串的底层实现会自动根据需要进行切换，无需我们操心，非常nice。

# 对象之——列表
2种实现： `ziplist`,`linkedlist`
压缩列表和链表。
{% img /images/2019-01/ziplist.png 800 1200 ziplist %}
{% img /images/2019-01/linkedlist.png 800 1200 linkedlist %}
可以看出`ziplist`顾名思义是压缩链表，申请了一大块连续空间来存放列表。
优点：
- 对缓存友好；
- 可以快速访问尾节点。
- 可以快速访问前一个节点(通过计算地址)。可以达到双向链表的效果。
缺点：不灵活。

使用 `ziplist` 编码的必要条件：
1. 所有字符串元素的长度<`64B`；(对应参数`list-max-ziplist-value `)
2. 元素数量< `512`个。(对应参数`list-max-ziplist-entries`)
任一不满足，使用普通`linkedlist`。

## ziplist
{% img /images/2019-01/ziplist_node.png 800 1200 ziplist_node %}
压缩列表的数据节点由`previous_entry_length`、 encoding 、 content 三个部分组成。
由于是连续空间，可以通过`previous_entry_length`知道前一个节点的长度，用当前地址减去这个长度即为前一个节点的地址，因此有双向链表的效果。

`previous_entry_length`:
1个字节: 不以0XFE开头，直接存放长度。(可见这个长度小于0XFE，也就是小于254)
5个字节: 以0XFE开头，去掉首部的0XFE后才是长度。

用`previous_entry_length`而不是直接存前一个节点地址对比:
1. 依赖连续空间;
2. 存地址起码4B,64位机器是8B，空间开销大于1B和5B. 而且连续空间下如果存地址，插入节点后所有地址都更改了。

### previous_entry_length引起的连锁更新
由于`previous_entry_length`可能是1B或者5B，如果所有节点原先都是253B，长度字段都只需1B。这个时候如果第一个节点变长为255B，则后一个节点的长度字段变长为5B、变长了4B，总长度变为257B。然后，再后一个节点存放的长度也要变化。依次类推，连锁影响后面所有节点的长度字段，引发连锁更新。

# 对象之——哈希表
{% img /images/2019-01/hash_ziplist.png 800 1200 hash_ziplist %}

哈希对象的编码可以是`ziplist` 或者 `hashtable` 。
转换参数:
`hash-max-ziplist-value`: 默认64B;
`hash-max-ziplist-entries `: 默认512。

# 对象之——集合
集合对象的编码可以是 `intset`(整数集合) 或者 `hashtable` 。

# 对象之-有序集合
有序集合的编码可以是 `ziplist` 或者 `skiplist` 。
{% img /images/2019-01/zset_ziplist.png 800 1200 zset_ziplist %}

## skiplist
```c
typedef struct zset {
    zskiplist *zsl; // 保存实际元素和score     服务的命令: ZRange/ZRank (O(LogN))
    dict *dict;     // 保持(元素=>score)的映射 服务的命令: ZSCORE      （O(1)）
} zset;
```
`ZSet`的成员: `(Obj: String,Score: Double)`
本来是两倍存储空间(zsl+dict)，但因为用指针，只有指针地址两倍了，实际数据还是1倍。
(数据在跳表，dict里只有指针。先插入跳表，再保存到dict。)

## skiplist和ziplist切换参数
`zset-max-ziplist-entries`: 元素数量,默认128.
`zset-max-ziplist-value`:   元素大小,默认64B。

一些操作复杂度:
`ZCARD`: O(1). 直接有存长度。
`ZCOUNT key min max`： logN+M,索引到min以后一直读到max.这里的min和max是分值。
`ZRANGE key start stop [WITHSCORES]`: logN+M,这里的start,stop是分值的排名。
`ZRANGEBYSCORE key min max [WITHSCORES] [LIMIT offset count]`: logN+M,分值范围。
`ZRANK key member`: logN, 返回排名。
`ZREM key member [member ...]`: logN,删除元素


# 共享对象
Redis 只对包含整数值的字符串对象进行共享.
整数范围配置:
```
redis.h/REDIS_SHARED_INTEGERS 10000 
```
所以默认共享0 到 9999 的字符串对象。


查看对象的引用计数:
```
OBJECT REFCOUNT A
```

