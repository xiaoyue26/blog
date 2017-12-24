---
title: redis笔记
date: 2016-04-03 20:00:39
tags: redis
categories:
- redis

---


## 1. 安装:
mac下安装很简单:
```
brew install redis
```
启动服务端:
```
redis-server
```
启动客户端:
```
redis-cli
```

## 2. 配置
mac默认的配置文件在:
```
/usr/local/etc/redis.conf
```
可以通过命令`brew info redis`查看到. 
但具体的安装路径,找了一圈连接发现是在:
```
/usr/local/Cellar/redis
```
cli会话下查询某项配置的值:
```
CONFIG GET loglevel
CONFIG GET * # 获取所有
```

## 3. 数据类型
Redis支持五种数据类型：string（字符串），hash（哈希），list（列表），set（集合）及zset(sorted set：有序集合)。

#1. string
>redis 127.0.0.1:6379> SET name "runoob"
OK
redis 127.0.0.1:6379> GET name
"runoob"

#2. hash(k,v都是string)
>127.0.0.1:6379> HMSET user:1 username runoob password runoob points 200
OK
127.0.0.1:6379> HGETALL user:1
1) "username"
2) "runoob"
3) "password"
4) "runoob"
5) "points"
6) "200"

# 3. list
Redis 列表是简单的字符串列表，按照插入顺序排序。你可以添加一个元素到列表的头部（左边）或者尾部（右边）。
> redis 127.0.0.1:6379> lpush runoob redis
(integer) 1
redis 127.0.0.1:6379> lpush runoob mongodb
(integer) 2
redis 127.0.0.1:6379> lpush runoob rabitmq
(integer) 3
redis 127.0.0.1:6379> lrange runoob 0 10
1) "rabitmq"
2) "mongodb"
3) "redis"

相关命令
```
lpush key value [value...]
rpush key value [value...]
lrange key start end #左闭右闭
```

# 4. Set
命令: ```sadd key member```
> redis 127.0.0.1:6379> sadd new_set redis
(integer) 1
redis 127.0.0.1:6379> sadd new_set mongodb
(integer) 1
redis 127.0.0.1:6379> sadd new_set rabitmq
(integer) 1
redis 127.0.0.1:6379> sadd new_set rabitmq
(integer) 0
redis 127.0.0.1:6379> smembers new_set
1) "rabitmq"
2) "mongodb"
3) "redis"

# 5. zset 排序的set,根据score排序从小到大
命令: ```zadd key score member ```
>redis 127.0.0.1:6379> zadd new_zset 0 redis
(integer) 1
redis 127.0.0.1:6379> zadd new_zset 0 mongodb
(integer) 1
redis 127.0.0.1:6379> zadd new_zset 0 rabitmq
(integer) 1
redis 127.0.0.1:6379> zadd new_zset 0 rabitmq
(integer) 0
redis 127.0.0.1:6379> ZRANGEBYSCORE new_zset 0 1000
1) "redis"
2) "mongodb"
3) "rabitmq"

## 连接远端redis服务器

```
redis-cli -h host -p port -a password
```
