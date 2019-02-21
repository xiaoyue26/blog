---
title: redis设计与实现笔记5-事件
date: 2019-02-21 21:53:58
tags:
- redis
categories:
- redis

---

redis服务器需要处理的事件可以分为两类：
1. 文件事件(file event): 套接字的抽象。服务器通过套接字与客户端或其他服务端连接、通信；
2. 时间事件(time event): 定时任务。

## 文件事件
事件模型是基于Reactor开发的:
{% img /images/2019-02/file_event_handler.png 800 1200 file_event_handler %}
1. 套接字;
2. IO多路复用程序；
3. 事件分派器;(dispatcher)
4. 事件处理器。

每当一个套接字准备好执行：
连接应答(accept)、
写入、
读取、
关闭
等操作时，就会产生一个事件（可能并发）。
{% img /images/2019-02/force_single.png 800 1200 force_single %}
`IO多路复用程序`：把上游并发的事件组织成一个队列（方便下游单线程地使用）。
当上一个套接字事件被处理完以后，IO多路复用程序才会向事件分派器传送下一个套接字事件。

### IO多路复用
具体啥是IO多路复用？就是一个进程处理多个连接。
方案有很多：(详见:http://xiaoyue26.github.io/2017/11/06/2017-11/epoll%E7%9B%B8%E5%85%B3%E5%9F%BA%E7%A1%80%E7%9F%A5%E8%AF%86/)
1. 循环、轮询；
2. Select;
3. poll;
4. epoll(红黑树)/kqueue(哈希表);
5. libevent库。

### 事件优先级
先处理可读，再处理可写。
### API接口
```
1. createFileEvent(套接字描述符,事件类型,事件处理器):开始监听;
2. deleteFileEvent(套接字描述符,事件类型): 取消监听;
3. getFileEvents(套接字描述符): 返回被监听的事件类型;
4. wait(套接字描述符,事件类型,超时时长(ms)): 等待事件;
5. apiPoll(超时时长): 等待所有被监听事件直至至少一个发生；
6. processEvent():等待事件,然后分派;
7. getApiName: 返回底层使用的IO库(epoll,poll或select等)
```

## 时间事件
1. 定时事件：指定时间后执行1次；
2. 周期事件：每隔指定时间就执行1次（总N次)。

### 时间事件属性：
1. id: 全局唯一，递增；
2. when: 毫秒，事件到达时间；
3. timeProc: 函数，到期执行。

定时事件和周期事件区分：
1. timeProc返回`AE_NOMORE`: no more事件，不再调用；
2. timeProc返回30： 周期事件，30ms后再次调用。

TODO: 全局唯一id=>服务器内唯一？

### 实现
{% img /images/2019-02/time_event.png 800 1200 time_event %}
所有时间事件放在一个无序链表中，每次遍历整个链表，查找所有已到达的时间事件。

### 性能
由于时间事件很少（1，2个），所以虽然实现很naive，性能也还行。

### 现有时间事件
`serverCron`:（每100ms）
1. 更新统计信息：时间、内存、数据库占用；
2. 清理过期KV；
3. 关闭清理失效客户端；
4. AOF\RDB持久化；
5. 主从同步；
6. 集群模式：定期同步、连接测试。


## 事件循环
{% img /images/2019-02/all_event.png 800 1200 all_event %}
事件无抢占。
先文件事件后时间事件，因此时间事件一般会滞后一点。

