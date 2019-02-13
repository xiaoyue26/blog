---
title: redis设计与实现笔记3-数据库
date: 2019-02-13 09:50:59
tags:
- redis
categories:
- redis

---


## 架构:
server-> db* -> dict*(k/v)
```c
typedef struct redisServer {
    // ...
    redisDb *db;
    int dbnum;    
    // ...
} redisDb;
```

# 数据库键空间
```c
typedef struct redisDb {
    // ...
    // 数据库键空间，保存着数据库中的所有键值对
    dict *dict;
    // 过期时间:
    dict *expires;
    // ...
} redisDb;
```

db默认有16,客户端可以通过`select 1`命令来选择1号数据库。
每个client会保存当前在哪个db的状态变量。

# 键:Key
key只能为字符串。
整个架构最多三层: db->hashtable->value
{% img /images/2019-02/db_kv.png 800 1200 db_kv %}

# key过期
## 过期删除策略
1. 定时统一删除 (一般不用，卡太久)
2. 惰性删除(默认使用,访问时发现过期才删除)
3. 定期删除(默认使用,类似于增量删除,每次删除最近过期的)

## redis持久化策略
1. RDB: 定期(比如五分钟)做一次快照; (`Redis DataBase`)
2. AOF: 类似于WAL，存每条操作指令日志。(`Append only file`)

## RDB和AOF下的过期删除
1. RDB:  主服务器生成快照时删除过期key; 从库则保留。从库只有收到主库DEL指令时候才会删除过期key。
2. AOF:  保存DEL指令即可。

```
从库如果没有收到主库的DEL指令，即使已经key已经过期，也会返回给客户端。
```
主从复制要点：（中心化，一致性）
`从库`只接受`主库`的`写`指令，自己只执行`读`。


# 通知机制 
两个功能:
1. 订阅某个key的所有操作; (只能知道发生了什么类型的操作,不知道操作数)
2. 订阅某个库下的所有DEL操作.(或者别的什么操作)
换句话说就是订阅某个库下key，或者订阅指令。

