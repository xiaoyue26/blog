---
title: redis的embstr为什么是39B
date: 2019-01-19 10:47:50
tags:
- redis
categories:
- redis

---

回顾字符串的三种实现:

- `int`: long能存下的数会用数字编码;
- `embstr`: <=39B的字符串，或者浮点数，会用`embstr`编码;
- `raw`: >39B的字符串，会用`raw`编码,也就是简单动态字符串(`SDS`)。
{% img /images/2019-01/raw.png 800 1200 raw %}
{% img /images/2019-01/embstr.png 800 1200 embstr %}

`embstr`顾名思义就是嵌入式字符串，把元数据和实际数据存放在一起，好处是这样可以对缓存友好，可以减少一次寻址。坏处是只能支持短字符串。
这里为啥是39B作为切换条件呢?(而且没有看到参数进行调节)，这其实是存储结构决定的。

> embstr本质上是利用了元数据存储空间里的剩余空间存储字符串。

所以当元数据剩了40B，`embstr`就最多只能存39B了(还要留1B存`\0`)。
回顾元数据结构：

```c
#define LRU_BITS 24
typedef struct redisObject {    // redis对象
    unsigned type:4;    // 类型,4bit
    unsigned encoding:4;    // 编码,4bit
    unsigned lru:LRU_BITS; /* lru time (relative to server.lruclock) */ // 24bit
    int refcount;   // 引用计数
    void *ptr;  // 指向各种基础类型的指针
} robj;
typedef struct sdshdr {
    unsigned int len;
    unsigned int free;
    char buf[];
}sds; 
```
{% img /images/2019-01/sizeof0.png 800 1200 sizeof0 %}
{% img /images/2019-01/sizeof1.png 800 1200 sizeof0 %}

## redisObject占16B
`16B=(4bit+4bit+24bit)+4B+8B= (32/8)B+12B`

## sdshdr占8B
`8=4+4+0`
注意这里的`char buf[]`不占空间，当里面存放字符串的时候才占空间，而且末尾需要1B存放`\0`。

按最小分配64B来算，剩余空间就是64B-16B-8B-1B=39B。

但是，如果仔细看看这里`sdshdr`里的，两个4B的长度：
```c
unsigned int len;
unsigned int free;
```
想一下，这里最大长度也才39,根本不需要这么大的数据结构来存长度，因此其实还是有压榨空间的。


# 从39B到44B
新版本的`embstr`已经不是39B了，改成了44B。本质上是元数据的剩余空间再次发生了变化。
新版本的元数据(从`SDS_TYPE_5`到`SDS_TYPE_64`5个)：
(https://github.com/antirez/redis/blob/unstable/src/sds.h)
```c
/* Note: sdshdr5 is never used, we just access the flags byte directly.
 * However is here to document the layout of type 5 SDS strings. */
struct __attribute__ ((__packed__)) sdshdr5 {
    unsigned char flags; /* 3 lsb of type, and 5 msb of string length */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr8 {
    uint8_t len; /* used */
    uint8_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr16 {
    uint16_t len; /* used */
    uint16_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr32 {
    uint32_t len; /* used */
    uint32_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr64 {
    uint64_t len; /* used */
    uint64_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
}

```
其中`sdshdr5`其实没有用，用的是`sdshdr8`，因此这里的元数据结构从8B下降到3B，`embstr`可用空间也就从39B上升到44B了。
