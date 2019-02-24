---
title: redis设计与实现笔记7-复制
date: 2019-02-24 20:46:10
tags:
- redis
categories:
- redis

---

# 第15章 复制
redis服务器B执行slave of命令后可变成另一台服务器的从库：
```sh
127.0.0.1:12345> SLAVEOF 127.0.0.1 6379
# 12345端口就变成6379的从库了。
```

## 旧版复制功能的实现
1. 同步(`sync`): 更新从库状态；
2. 传播(`propagate`): 持续维持一致性。

### 同步
主库发RDB文件给从库，从库载入一下。

旧版缺点：
每次断线后，重新同步，也就是重新生成RDB文件，重新全量载入。

## 新版复制功能的实现(2.8+版本)
引入`PSYNC`命令，它有两种模式:
1. 完整重同步: 初次复制；
2. 部分重同步：断线重连。

### 部分重同步：
1. 主从库的复制偏移量;(replication offset)
2. 主库的复制积压缓冲区;(replication backlog):默认1MB，FIFO队列。
3. 服务器的运行id(run ID)。

如果重连时，需要的数据还在缓冲区，就部分同步；
如果重连时，需要的数据已经被删除，就完全同步。

所以缓冲区设置稍微大一些最好。

### 复制流程
1. 从库:记录主库地址端口等信息到内存；
2. 建立套接字连接；
3. 从库: PING 主库: PONG，确认连接健康；
4. 身份验证：(是否需要认证，密码)，两个维度都需要相同才能继续；
5. 从库=>主库: 请用xxx端口联系从库;
6. 同步： 主库从库互为客户端：
完全同步： 主库=>从库: 保存在缓冲区的写命令;
部分同步： 主库=>从库: 保存在复制积压缓冲区的写命令。
7. 命令传播： 主库=>从库: 新的写命令。

### 心跳检测
从库=>主库: 
```sh
REPLCONF ACK <replication_offset> # 从库当前复制偏移量
```
三个作用：
1. 主库确定各个从库的健康状态；
2. 检测命令丢失：主库检测从库有没有漏的复制，漏则重发；
3. `min_slave_to_write`参数：如果配置了这个，主库可以在从库太少的时候拒绝写命令。


# slaveof命令源码细节
从库确认收到主库的完整rdb文件后，才清空旧数据库。
(而不是说不分青红皂白上来就把自己清空了，那就太傻了。)
相关代码(https://github.com/huangz1990/redis-3.0-annotated/blob/unstable/src/replication.c):
```c
/* Check if the transfer is now complete */
    // 检查 RDB 是否已经传送完毕
    // 1036行: 
    if (server.repl_transfer_read == server.repl_transfer_size) {

        // 完毕，将临时文件改名为 dump.rdb
        if (rename(server.repl_transfer_tmpfile,server.rdb_filename) == -1) {
            redisLog(REDIS_WARNING,"Failed trying to rename the temp DB into dump.rdb in MASTER <-> SLAVE synchronization: %s", strerror(errno));
            replicationAbortSyncTransfer();
            return;
        }

        // 先清空旧数据库
        redisLog(REDIS_NOTICE, "MASTER <-> SLAVE sync: Flushing old data");
        signalFlushedDb(-1);
        emptyDb(replicationEmptyDbCallback);
```



