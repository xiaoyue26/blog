---
title: epoll相关基础知识
date: 2017-11-06 19:52:23
tags: 并发
categories:
- java
- 并发
---

# 背景:
C10K问题. 并发访问量>=10K => 维持TCP连接的进程数超过上限.

- 远古解决方案:
1. 用`UDP`. (腾讯QQ)
2. `select`模式. 每个TCP连接占用一个文件描述符, 所以进程能处理的连接最大不能超过1024. 
3. `poll`模式. 每次收到数据需要遍历每一个连接查看哪个连接有数据请求。

2002年左右开始的解决方案:
>`kqueue`: FreeBSD
`epoll`:  Linux (2.5.44内核)
`IOCP`: Windows

- epoll的编程模型: 
异步非阻塞回调. `Reactor`,事件驱动,事件轮询(`EventLoop`).

- 新的问题
`epoll`编程太繁琐,层层回调/嵌套.

- 解决方案:
1. 默默忍受: 性能最优.
2. 协程,`coroutine`: 需要解决某个协程密集计算带来的问题.

- 协程
封装了事件回调,底层库通过保存状态(代码行数,局部变量的值),替程序猿进行回调. 程序猿可以就像一切刚开始的时候那样只写仿佛同步阻塞的代码.

`go`语言中具体代码:
```go
go doSomething();
// 类似于 threadPool.submit(new Thread(new Runnable(){run(){doSomething();}}))的效果
```
会提交一个协程到go语言自带的协程池子里。由于go语言每个函数可以作为一个协程，所以语法显得非常简练方便，而且无需操心线程池。
此外`go`中`main`函数也是一个协程，而且不是守护的。（java中main函数是守护的,会等待所有其他线程运行结束）
协程的同步依然需要自己管理，一般是用channel(管道)进行。
语法上,协程提供了一个写非阻塞操作的非常简洁的关键字(`go`)。
实现上,协程提供了非常方便的协程池,而且由于是用户态线程,因此每个协程的开销足够小,小到可以将每个函数这么小的粒度作为协程。 


缺陷: 
由于实现上是使用用户态线程/进程,当某一个协程中有密集计算时,其他协程就不运行了. 每个线程上装配的协程的切换是需要在每次协程中发生中断时进行判断的。

解决方案:
- `GoLang`: 用户在密集计算的代码中自己`yield`. 
- `ErLang`: 无需关心. 由于底层有`ErLang`自己开发的VM,会自动检测并进行切换.(`rabbitmq`)


# 具体实现
- 问题的本质:
一个进程处理一个连接, 导致进程开销太大. 
因此解决思路就是一个进程处理多个连接.也就是:
`IO多路复用`. 

实现1:
> 每个进程循环处理多个连接.
缺点: 其中一个连接处理卡住会阻塞整个应用. 

## select模式
实现2: (`select`模式)
> 维护一个fd_set状态结构体.
阻塞的原因是IO资源的争用,如某个文件句柄是否可读,可写.(锁)
告诉内核该进程关心的文件句柄, 让内核检查多个文件句柄.

进程发送句柄列表=>内核逐个检查状态是否可用. 

缺陷:
> 1. 每个进程有关注的文件句柄上限;
2. 需要重复初始化关心的文件句柄结构体;
3. 逐个排查连接效率低.

##  poll模式
实现3:(`poll模式`)
> 设计新的数据结构.
通过pollfd数组向内核传递需要关注的事件: 消除文件句柄上限;
不同字段区分关注事件和发生事件: 避免重复初始化.

缺陷:
> 1. 逐个排查连接效率低.

## epoll模式
实现4: (`epoll`模式)
内核升级到能对每个FD注册回调,进程能够直接知道是哪个句柄状态变化,而不用轮询,监听回调就行了.
(linux 2.5.44新增`api`)
事件模型.

## epoll具体实现细节
**架构:**
```
// 一堆事情(IO访问),需要切换到内核态访问相应地址空间.例如把数据从内核地址空间拷贝到用户地址空间. 拷贝完以后触发一个读可用事件.
用户进程->内核(
    readyList
   ,红黑树<文件描述符(如Socket)>
)
```
**工作流程:**
```
1. 用户进程调用epoll_create;
2. 内核创建readyList,红黑树;
3. 用户进程调用epoll_ctrl,传递监控的句柄(如Socket),以及在上面关注的事件;
4. 内核将句柄插入红黑树;
5. 内核的中断处理程序注册一个回调,如果红黑树中某个句柄的中断到了,
把它对应的事件放到ReadyList. 

--- 另一个流程 --- 
1. 用户进程调用epoll_wait
2. 内核从ReadyList返回可回调的事件. 

```

**LT模式与ET模式**
LT模式: 水平触发(JAVA NIO默认),一个句柄上的事件一次没处理完,会再下次调用epoll_wait的时候依然返回它;
ET模式: 边缘触发,仅在第一次返回.

LT模式:
```
1. 用户进程调用epoll_wait;
2. 内核从ReadyList返回数据;// 就绪的Socket拷贝到用户态内存
3. 清空ReadyList;
4. 检查Socket句柄,如果有未处理完的事件(数据没读完),将句柄加入ReadyList.
```
ET模式没有上述的第四步.

**LT模式注意事项**
> 不需要写的话,不要关注写事件.
长期关注写事件会导致CPU100%问题. 

**ET模式注意事项**
> 每次都要读到返回EWOULDBLOCK为止. (要读干净)

**epoll为什么使用红黑树**
因为epoll要求快速找到某个句柄,因此首先是一个Map接口,候选实现:
1. 哈希表 O(1)
2. 红黑树 O(lgn)
3. 跳表 近似O(lgn)
据说老版本的内核和FreeBSD的Kqueue使用的是哈希表. 

个人理解现在内核使用红黑树的原因:
1. 哈希表. 空间因素,可伸缩性.
(1)频繁增删. 哈希表需要预估空间大小, 这个场景下无法做到.
间接影响响应时间,假如要resize,原来的数据还得移动.即使用了一致性哈希算法,
也难以满足非阻塞的timeout时间限制.(时间不稳定)
(2) 百万级连接,哈希表有镂空的空间,太浪费内存.
2. 跳表. 慢于红黑树. 空间也高. 
3. 红黑树. 经验判断,内核的其他地方如防火墙也使用红黑树,实践上看性能最优.


**数据结构:**
```c
typedef union epoll_data {
     void *ptr;
     int fd; 
     __uint32_t u32;
     __uint64_t u64; 
}epoll_data_t;  // <=

struct epoll_event { 
    __uint32_t events; /* Epoll events */ 
    epoll_data_t data;/* User data variable */
 };
```
C库的API:
```c
int epoll_create(int size); // size: 最大句柄数
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);  
int epoll_wait(int epfd, struct epoll_event *events,int maxevents, int timeout);
```
具体语义:
```c
epoll_create: 让内核创建一个epoll item;
epoll_ctl: 修改epoll item: 增加socket句柄,或删除socket句柄,注册回调事件;
epoll_wait: 等待注册的事件发生. 
传入的参数epoll_event会从内核那捞回结果.(被回调的事件)
```


## 实现5: libevent库
封装`epoll`. 统一不同平台api. 
使用方式:
```
1. 初始化事件event,定义回调函数f;
2. 将事件加入系统监控事件列表;
3. 监听事件及分发.
```

**NIO的epoll bug**
部分Linux的2.6的kernel中，poll和epoll对于突然中断的连接socket会对返回的eventSet事件集合置为POLLHUP，也可能是POLLERR，eventSet事件集合发生了变化，这就可能导致Selector会被唤醒。
进一步导致jdk中NIO实现中某段代码的死循环.

Netty的解决方案:
1. 将select返回值为0进行计数,直到大于阈值`EPOLL_BUG_WORKAROUND`;
2. 重建Selector. 



# C10M问题
1. 数据包经过linux内核协议栈导致性能巨大下降.
解决方案: 
(1)intel的DPDK框架. 直接接管网卡收到的数据包传递到业务代码中进行处理.

2. 存储访问时间消耗. 切换消耗:
解决方案:
(1) 重新设计操作系统, 包括内存池,cache,内存分页,cpu管理等等.

# 术语/概念
## 缓存IO
标准IO. 操作系统先将IO数据缓存到文件系统的页缓存(page cache)中,然后再从操作系统内核的缓冲区拷贝到应用程序的地址空间. 
缺点:
重复拷贝带来很大的CPU/内存开销. 

- 阻塞/非阻塞: 
> 内核会不会立即回答;

- 同步/异步: 
> 进程会不会等内核回答. 


## IO模式
1. 阻塞IO
2. 非阻塞IO
3. IO多路复用
4. 信号驱动IO
5. 异步IO

## 阻塞IO
1. 进程请求IO数据;
2. 进程阻塞;
3. 内核准备数据,拷贝到内核空间;
4. 拷贝到进程地址;
5. 解除阻塞.

## 非阻塞IO
- 轮询
1. 进程请求IO数据;
2. 内核回答不OK;
3. 进程不断轮询;
4. 内核终于准备好了数据,回答OK. 

## IO多路复用 
包括: select,poll,epoll
> 一次轮询多个socket,任何一个socket数据搞定的话,内核都会回复一次. 


## 异步非阻塞IO
类似于非阻塞, 取消了轮询. 
进程不等待内核的回复.
内核完成准备数据后会通知进程.


