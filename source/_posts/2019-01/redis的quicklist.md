---
title: redis的quicklist
date: 2019-01-19 20:29:18
tags:
- redis
categories:
- redis

---

列表对象: `REDIS_LIST`

列表对象以前有两种编码: `linkedlist`和`zipList`：
`linkedList`: 双向链表,灵活扩展,存取效率低。
`zipList`: 连续存储,存取效率高，对缓存友好，扩展性差，修改插入需要级联移动.甚至可以粗略看做数组。
小数据量的时候会使用`zipList`,大数据量的时候会使用`linkedList`。
但它们其实是两个极端，各有优缺点。因此之后的版本就把它们混血了一下，出了一个`quickList`，平衡优缺点。

## quickList设计
粗略地看，把双向链表的每一个节点变成`zipList`，就是`quickList`了。
在原来的两个数据结构之间做一个折中，量化折中的参数:
```
# 负数表示存储容量等级,每个节点上的ziplist大小不能超过8Kb（也就是2^3KB，以此类推）
list-max-ziplist-size -2
# 正数表示存储个数，每个节点上的zipList内元素不能超过5个:
list-max-ziplist-size 5
```

还有一个压缩参数:
```
# 不进行压缩:
list-compress-depth 0 
# 除了首尾1个，中间的压缩:
list-compress-depth 1
```
由于双向链表一般用`LPush`,`Rpush`，两端的数据访问比较频繁，

## quickList源码
```c
typedef struct quicklist {
    quicklistNode *head;    // 头结点
    quicklistNode *tail;    // 尾节点
    unsigned long count;    // 所有数据的数量
    unsigned int len;       // quicklist节点数量
    int fill:16;          // 单个ziplist的大小限制
    unsigned int compress:16;   // 压缩深度
} quicklist;
typedef struct quicklistNode {
    struct quicklistNode *prev; // 前一个节点
    struct quicklistNode *next; // 后一个节点
    unsigned char *zl;  // ziplist
    unsigned int sz;             // ziplist的内存大小
    unsigned int count:16;     // zpilist中数据项的个数
    unsigned int encoding:2;   // 1为ziplist 2是LZF压缩存储方式
    unsigned int container:2;  
    unsigned int recompress:1;   // 压缩标志, 为1 是压缩
    unsigned int attempted_compress:1; // 节点是否能够被压缩,只用在测试
    unsigned int extra:10; /* more bits to steal for future usage */
} quicklistNode;
```

