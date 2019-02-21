---
title: redis设计与实现笔记6-客户端与服务器
date: 2019-02-21 21:55:54
tags:
- redis
categories:
- redis

---

# 客户端
服务端保存的客户端状态:(`redisClient`)
1. 套接字;
2. 客户端名字
3. 标志值flag;
4. 正在使用的数据库指针\号码;
5. 客户端当前要执行的命令、参数..;
6. 客户端输入、输出缓冲区；
7. 复制状态信息；
8. 事务状态；
9. 身份验证标志：0未通过；1：已经通过身份验证；
10. 创建时间、最后一次通信时间；
11. 其他。

```c
struct redisServer{
    // 所有客户端状态的链表
    list *clients;
}
```

## 相关命令
列出客户端:
```shell
127.0.0.1:6379> client list
id=489 addr=127.0.0.1:57480 fd=5 name= age=6151 idle=582 flags=N db=0 sub=0 psub=0 multi=-1 qbuf=0 qbuf-free=0 obl=0 oll=0 omem=0 events=r cmd=get
id=490 addr=127.0.0.1:38584 fd=6 name= age=15 idle=0 flags=N db=0 sub=0 psub=0 multi=-1 qbuf=0 qbuf-free=32768 obl=0 oll=0 omem=0 events=r cmd=client
```

## 套接字fd
fd=-1: 伪客户端:加载AOF或执行Lua脚本；
fd>-1: 普通客户端。

## 名字(name)
默认都没有名字，可以使用`client setname`命令设置。

## 客户端关闭
关闭的原因：
1. 客户端进程退出、被杀死(`Client kill`)，网络连接断开；
2. 客户端发送了不符合协议格式的命令，被关闭；
3. 空转时间超过`timeout`配置;
4. 发送命令大小超过输入缓冲区限制；
5. 回复命令大小超过输出缓冲区限制。

输出缓冲区需满足的两个限制：
1. 硬性限制：超出硬性限制大小，立即关闭；
2. 软性限制：最多可以超出软性限制持续时长xxx秒。

示例配置：
```shell
# 普通客户端硬性限制和软性限制都不限制:
client-output-buffer-limit normal 0 0 0
# 从服务器客户端硬性限制为256MB,软性限制为64MB、60秒：
client-output-buffer-limit slave 256mb 64mb 60
# 执行发布与订阅功能的客户端：硬性限制32mb,软性限制8mb 60秒:
client-output-buffer-limit 32mb 8mb 60
```

# 服务端
## 命令的执行流程
1. 用户=>客户端: 输入命令；
2. 客户端=>服务端: 命令按通信协议传输到服务器输入缓冲区；
3. 服务端: 等待可读事件发生后，读输入缓冲区，解析命令；
4. 服务端：执行命令，将回复写入输出缓冲区；
5. 服务端=>客户端：等待可写事件发生后，从输出缓冲区传输给客户端。
6. 客户端=>用户: 回显命令结果。


