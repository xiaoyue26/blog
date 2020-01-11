---
title: Redis设计与实现笔记1-数据结构
date: 2019-01-12 19:55:22
tags:
- redis
categories:
- redis

---

参考\搬运资料: http://redisbook.com/index.html


# 设计哲学
设计简单优于巧妙实现:

| 表现        | 原因   |   
| :--------:   | :----- |  
| 不处理循环引用 | 不产生循环引用。数据结构足够简单，最多2层。  | 
| 不处理事务回滚 | 程序员自己处理代码错误|
换个角度就是，没把握做好的事情就干脆不做了。
解决问题的方法就是不产生问题。不写错代码，很科学，很彻底XD。

## 简单动态字符串(simple dynamic string，SDS)
```c
struct sdshdr {
    // 记录 buf 数组中已使用字节的数量
    // 等于 SDS 所保存字符串的长度
    int len;
    // 记录 buf 数组中未使用字节的数量
    int free;
    // 字节数组，用于保存字符串
    char buf[];//\0结尾，不计入len
};
```
类似`head-body`结构的消息。比`\0`分割的C字符串简单易用多了，性能更高，代价是付出了更多的内存。
之所以最后用上了`\0`结尾,是为了重用一部分c库里的字符串库，进一步降低了使用成本。

### 性能优势点:
1. 获取字符串长度: O(1)
2. 空间预分配，减少系统调用。
3. 惰性空间释放，减少系统调用。

### 安全性优势点:
1. 有效防止缓冲区溢出
2. 二进制安全，可以保存\0。

### 缺点：
1. 如果有大量字符串缩短，内存泄露严重。

### sdscat
字符串拼接：
{% img /images/2019-01/sds.png 800 1200 sds %}
计算拼接后总长度len，扩展空间为2*len，拼入字符串。最后一定会有一半free空间。

#### 空间预分配
假如现有空间不足以存放字符串：
- 总长度len<1MB: 总空间为`2*len+1`;
- 总长度len>=1MB: 总空间为`len+1MB+1`。
换句话说，预分配的空间上限是1MB，尽量为len。
这么做的目的是减少空间申请、回收的系统调用性能开销。

### 惰性空间释放
 字符串缩短的时候，默认不进行内存回收（可能内存泄露）。
 需要手动调API回收。
 
 
# 链表(RPush)
单个节点：
```c
typedef struct listNode {
    // 前置节点
    struct listNode *prev;
    // 后置节点
    struct listNode *next;
    // 节点的值
    void *value;
} listNode;
```
可以看出是一个简单的双向链表。
{% img /images/2019-01/double_linked_list.png 800 1200 double_linked_list %}
整个列表容器：
```c
typedef struct list {
    // 表头节点
    listNode *head;
    // 表尾节点
    listNode *tail;
    // 链表所包含的节点数量
    unsigned long len;
    // 节点值复制函数
    void *(*dup)(void *ptr);
    // 节点值释放函数
    void (*free)(void *ptr);
    // 节点值对比函数
    int (*match)(void *ptr, void *key);
} list;
```
 
# 哈希表（HSet）
{% img /images/2019-01/hash_table.png 800 1200 hash_table %}
单个节点:
```c
typedef struct dictEntry {
    // 键
    void *key;
    // 值
    union {  // 三者取其1
        void *val;
        uint64_t u64;
        int64_t s64;
    } v;
    // 指向下个哈希表节点，形成链表
    struct dictEntry *next;
} dictEntry; 
```
整个表:
```c
typedef struct dictht {
    // 哈希表数组
    dictEntry **table;
    // 哈希表大小
    unsigned long size;
    // 哈希表大小掩码，用于计算索引值
    // 总是等于 size - 1
    unsigned long sizemask;
    // 该哈希表已有节点的数量
    unsigned long used;
} dictht;
```
对外接口:
```c
typedef struct dict {
    // 类型特定函数
    dictType *type;
    // 私有数据
    void *privdata;
    // 哈希表,有两个，第二个扩容时候用
    dictht ht[2];
    // rehash 索引
    // 当 rehash 不在进行时，值为 -1。 扩容时候用。
    int rehashidx; /* rehashing not in progress if rehashidx == -1 */
} dict;
```
dictType:
```c
typedef struct dictType {
    // 计算哈希值的函数
    unsigned int (*hashFunction)(const void *key);
    // 复制键的函数
    void *(*keyDup)(void *privdata, const void *key);
    // 复制值的函数
    void *(*valDup)(void *privdata, const void *obj);
    // 对比键的函数
    int (*keyCompare)(void *privdata, const void *key1, const void *key2);
    // 销毁键的函数
    void (*keyDestructor)(void *privdata, void *key);
    // 销毁值的函数
    void (*valDestructor)(void *privdata, void *obj);
} dictType;
```


sizemask为3的话，也就是0x11，就是说取最后两位作为哈希序号，也就是0~3，4个数。
所以此时hash表长度为4。以此类推，sizemask有n位,哈希表长度就有2^n。

## 渐进式hash
具体还是要参见: http://redisbook.com/preview/dict/rehashing.html
{% img /images/2019-01/rehash.png 800 1200 rehash %}

逐步把ht[0]中的数据迁移到ht[1]中。`rehashidx`为0的时候，表示0号位置已经全部迁往ht[1],依次类推，直到全部迁完。
交换ht[0],ht[1]以后，`rehashidx`改为-1.
在此过程中ht[0]不会有新插入，新请求全部去ht[1]。

### 扩容/rehash的触发
由时间事件serverCron根据负载因子来决定是否进行扩容。
扩容的实际执行是由1个专门的单线程来进行; (类似cms一样高响应,一次resh一个桶,尽量不阻塞线上读写请求)
相比之下,concurrentHashMap的扩容则由多线程进行,阻塞所有读写,高吞吐低响应。


# 跳表
{% img /images/2019-01/skiplist.png 800 1200 skiplist %}

有序集合使用。(ZSET)
节点源码: 
```c
typedef struct zskiplistNode {
    // 后退指针
    struct zskiplistNode *backward;
    // 分值
    double score;
    // 成员对象
    robj *obj;
    // 层
    struct zskiplistLevel {
        // 前进指针
        struct zskiplistNode *forward;
        // 跨度
        unsigned int span;
    } level[];
} zskiplistNode;
```
跳表容器:
```c
typedef struct zskiplist {
    // 表头节点和表尾节点
    struct zskiplistNode *header, *tail;
    // 表中节点的数量
    unsigned long length;
    // 表中层数最大的节点的层数
    int level;
} zskiplist;
```

从高层向低层找:
1. 在找的过程中用update[]数组记录每一层插入位置的上一个节点，
2. 用rank[]数组记录每一层插入位置的上一个节点在跳跃表中的排名。
3. 根据update[]插入新节点，插入完成后再根据rank[]更新跨度信息。

# 整数集合
整数集合（intset）是集合键的底层实现之一： 当一个集合只包含整数值元素， 并且这个集合的元素数量不多时， Redis 就会使用整数集合作为集合键的底层实现。

```redis
SADD numbers 1 3 5 7 9
OBJECT ENCODING numbers # 查看底层实现 "intset"
```
## 升级
默认用最小数据类型，新数据太大的话，会对整个集合数据类型升级，并且移动全部元素。
因此最坏时间复杂度为O(n)。

升级后不会降级。


# 压缩列表（ziplist）
压缩列表（ziplist）是列表键和哈希键的底层实现之一。
适用条件: 整数、小字符串。

```redis
RPUSH lst 1 3 5 10086 "hello" "world"
OBJECT ENCODING lst # 底层结构: ziplist
```
