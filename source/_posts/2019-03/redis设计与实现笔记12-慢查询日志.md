---
title: redis设计与实现笔记12-慢查询日志与监视器
date: 2019-03-18 09:20:07
tags:
- redis
categories:
- redis

---

# 第23章 慢查询日志
两个配置：
```
slowlog-log-slower-than: 超过多少微秒的命令记录到慢查询日志；
slowlog-max-len: 最多保存多少条慢查询日志。(删除最旧的，FIFO)
```

## 查看慢查询日志
```shell
slowlog get
1) 1) (integer) 4 # 日志id: uid
   2) (integer) 1578781447 # 日志执行时间戳
   3) (integer) 13 # 执行了多少微秒
   4) 1) "SET"     # 具体命令
      2) "database"
      3) "Redis"
...
```

## 慢查询日志存储
```c
struct redisServer{
    // 下一条慢查询日志的id:(类似于自增id)
    long long slowlog_entry_id;
    // 保存了所有慢查询日志的链表:
    list *slowlog;
    // 配置:
    long long slowlog_log_slower_than;
    unsigned long slowlog_max_len;
}

typedef struct slowlogEntry{
    long long id;
    time_t time;
    long long duration;
    robj **argv; // 命令和参数
    int argc; // 命令和参数数量
    
}
```

# 第24章 监视器
Monitor命令:
```shell
redis> MONITOR
```
然后客户端可以监视服务器当前处理的命令请求。
{% img /images/2019-03/monitor.png 800 1200 monitor %}

## 监视器实现
```c
struct redisServer{
    // 监视器链表:
     // 链表，保存了所有从服务器，以及所有监视器
    list *slaves, *monitors;    /* List of slaves and MONITORs */
}
```



## 这本书戛然而止
