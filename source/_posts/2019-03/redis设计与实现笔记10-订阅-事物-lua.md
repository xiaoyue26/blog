---
title: 'redis设计与实现笔记10-订阅,事物,lua'
date: 2019-03-10 20:08:34
tags:
- redis
categories:
- redis

---

# 发布订阅
(书上第18章)
客户端可以订阅某个频道，或者订阅符合某种模式的频道们。
{% img /images/2019-03/publish.png 800 1200 publish %}

## 订阅关系保存
```c
struct redisServer{
    // 保存所有频道的订阅关系:
    dict* pubsub_channels;
    // key: 频道名字(string)
    // value: 订阅的客户端们(链表)
    
    // 模式订阅: 订阅符合某种模式的频道
    dict* pubsub_patterns;
}
```

（如果有个客户端疯狂乱订阅，服务器是不是内存就爆了？）

# 第19章-事务
相关命令:
```sh
MULTI: 类似于事务开始start transaction;
# 中间一堆正常redis set命令。
EXEC: 类似于commit.
WATCH: 乐观锁，在exec执行前监视一些key是否被修改。
如果被修改，则拒绝执行事务（类似于CAS）

```
事务执行阶段不会执行别的客户端的命令。（相当于独占了）

### 事务开始
MULTI: 客户端状态打开`REDIS_MULTI`标识。

### 命令入队
除了`EXEC`,`DISCARD`,`WATCH`,`MULTI`之外的命令放入队列中；
否则立即执行。

### Watch功能实现
```c
typedef struct redisDb{
    // 正在被watch监视的key:
    dict *watched_keys;
    // key: 被监视的key
    // value: 监视的客户端们(list)
}redisDb;
```
存在redisDb中，可见每个数据库都保存一个这个字典。

每个修改操作都要检查watched_keys字典，通知对应的客户端。（打脏标记`REIDS_DIRTY_CAS`）

## 事务的ACID
A: 原子性 Atomicity
C: 一致性 Consistency
I: 隔离性 Isolation
D: 耐久性 Durability
### A：原子性

- mysql：
要么一个都不执行成功，要么都全部执行成功。

redis这里略有修改，它只保证执行，不保证执行成功：
要么一个都不执行，要么都全部执行。
（redis只检查编译错误，如命令不存在，不检查运行时错误）
redis执行事务过程中出错的话，不会回滚已经执行的命令。
（开发者表示这算程序员自己的锅）

### C: 一致性
单实例：肯定一致。
主从： 由raft保证；
cluster: 分slot以后，相当于单实例+主从，因此一致。

### I: 隔离性
也就是让并发执行达到串行一致性。
由于redis本来就是单线程串行执行事务，因此天然不需要做额外的事就能达到隔离性。

### D: 耐久性
redis三种模式:
无持久化存储: 无耐久
RDB: 不能完全保证；
AOF: `appendfsync`为`always`时，达到事务耐久性。


# 第20章-Lua脚本
主要涉及两个命令`EVAL`和`EVALSHA`。
redis服务器端2.6开始有lua环境，因此客户端可以：
1. 执行lua脚本: 
```shell
redis> EVAL "return 'hello world'" 0
```
最后的0表示输入参数的个数是0个。参见：http://doc.redisfans.com/script/eval.html
2. 通过SHA1校验和，执行对应的lua脚本:
```shell
redis> EVALSHA "a27e72..........."
```
这个校验和需要服务器认识才行，服务器认识的方法：
(1)服务器以前执行过对应的lua脚本；
(2)客户端用`SCRIPT LOAD`命令告诉过服务器：`SCRIPT LOAD "return 2*2"`。

### 如果在cluster模式：
lua脚本如果要使用redis数据库中的键，一定要通过参数传递进去，才能被分析出来，方便兼容新版本的集群功能。

### redis的lua环境
为了保证lua脚本之间不会互相影响，redis服务器需要保证luz脚本无副作用，它做了一下措施：
1. 修改随机数函数，消除副作用；
2. 禁止lua脚本创建全局变量；

但是好像遗漏了lua脚本对于已有全局变量的修改：
```shell
math.randomseed(10086) --change seed
```
应该是把这块儿交给程序员自行保证。


### lua脚本特有的排序辅助
此外，为了获得确定性一致的结果，redis对集合的输出结果做了排序。
例如调用`SMEMBERS`后的结果，会经过排序辅助函数进行排序。
保证同样的数据集的输出结果相同。


## lua_scripts字典
```c
struct redisServer{
    // 整个服务器全局的lua校验和
    dict *lua_scripts;
    // key: checksum
    // value: lua脚本代码
}
```
所有执行过或要求记住的lua校验和都会存下来。

## EVAL命令实现
3个步骤:
1. 计算校验和，然后用校验和定义一个函数f_校验和;
2. <校验和，脚本>保存到`lua_scripts`字典；
3. 执行函数。

比如校验和是a0e1ffff,函数名就是f_a0e1fffffffff。
然后利用函数的局部性，避免全局变量。

## 超时检测
参数`lua-time-limit`。
lua脚本的运行时间是有上限的，避免编程错误的死循环。
