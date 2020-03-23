---
title: redis大key问题
date: 2020-03-23 14:22:37
tags:
- redis
categories: 
- redis

---

0. 为啥不能有大key;
1. 有一些方法，避免大key;
2. 有大key，安全删除大key;

# what： 什么是大key问题
就是一个key的value特别大，比如一个hashmap中存了超多k,v;
或者一个列表key中存了超长列表，等等；
多大算大： hashmap中有100w的k,v => 1s延迟；
删除大Key的时间复杂度: O(N), N代表大key里的值数量，因为redis是单线程一个个删。
所以删大key也会卡qps。

## 开发规范
### 单key大小
Redis限制每个String类型value大小不超过512MB， 实际开发中，不要超过10KB，否则会对CPU和网卡造成极大负载。 hash、list、set、zset元素个数不要超过5000。

理论上限: 每个hashset里头元素数量< 2^32. 

### key的数量
官方评测： 单实例2.5亿
理论上限: 32位，2^32。约40亿

## 测试删除大key
可以用`slowlog`命令来查看删除耗时:
```lua
DEL big_key1
SLOWLOG GET 2
```


# why: 为啥不能有大key
redis的基础假设是每个操作都很快，所以设计成单线程处理；
所以如果有大key，基础设计就不成立了，会阻塞；

问题： 
1. 数据倾斜，部分redis分片节点存储占用很高；
2. 查询突然很慢，qps降低；


# How: 如何避免大key

分治法，加一些key前缀\后置分解（如时间、哈希前缀、用户id后缀）;

# 安全删除大key
1. 首先要找到大key才能删除;
2. 如何删除；

## 找到大key、删除大Key
### 当版本<4.0
 1、导出rdb文件分析: `bgsave`, `redis-rdb-tool`;
 2、命令: `redis-cli --bigkeys`,找出最大的key；
 3、自己写脚本扫描;  
 4、单个key查看: `debug object key`： 查看某个key序列化后的长度，每次看1个key的信息,比较没效率。

### 删除大Key:
分解删除操作：
list: 逐步ltrim;
zset: 逐步zremrangebyscore;
hset: hscan出500个，然后hdel删除；
set: sscan扫描出500个，然后srem删除；
依次类推；

### 当版本>=4.0

#### 寻找大key
命令: `memory usage`

#### 删除大key： lazyfree机制
`unlink`命令：代替DEL命令；
会把对应的大key放到`BIO_LAZY_FREE`后台线程任务队列，然后在后台异步删除；

类似的异步删除命令:
```lua
flushdb async: 异步清空数据库
flushall async: 异步清空所有数据库
```
异步删除配置:
```conf
slave-lazy-flush: slave接受完rdb文件后，异步清空数据库；
lazyfree-lazy-eviction: 异步淘汰key;
lazyfree-lazy-expire:   异步key过期;
lazyfree-lazy-server-del: 异步内部删除key；生效于rename命令
## rename命令: RENAME mykey new_name 
## 如果new_name已经存在，会先删除new_name，此时触发上述lazy机制

```
