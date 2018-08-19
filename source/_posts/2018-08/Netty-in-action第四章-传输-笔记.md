---
title: Netty in action第四章-传输-笔记
date: 2018-08-19 20:07:49
tags: 
- java
- netty
categories:
- java
- netty

---


# Netty通用api
`Netty`为`oio`/`nio`的传输实现封装了一个通用`api`。
方便用户低成本切换。
代码见:
https://github.com/xiaoyue26/netty-in-action-cn/blob/ChineseVersion/chapter4/

使用`netty`的话,阻塞和非阻塞实现的切换只需两行:
```java
// oio
NioEventLoopGroup group = new NioEventLoopGroup();
b.group(group).channel(OioServerSocketChannel.class)

// nio
NioEventLoopGroup group = new NioEventLoopGroup();
b.group(group).channel(NioServerSocketChannel.class)
```
也就是把`EvenLoopGroup`和`SeverSocketChannel`的实现换一下。


## Channel接口
{% img /images/2018-08/4.1.jpg 400 600 Channel接口 %}
`Channel`接口的父接口包括:
> Comparable<Channel>: 每个Channel是独一无二的,不可以相等,但是可以排序;
AttributeMap: 获得各种属性

依赖的接口:
> ChannelPipeLine: 有个方法返回它的pipeline;// 实现上大多有这个成员
ChannelConfig: 有个方法返回它的config;

子接口:
> ServerChannel
AbstractChannel: 如果引用不相等，保证当hash值相同时,CompareTo方法会抛出Error。

相关代码：
```java
@Override
    public final int compareTo(Channel o) {
        if (this == o) {
            return 0;
        }
        long ret = hashCode - o.hashCode();
        if (ret > 0) {
            return 1;
        }
        if (ret < 0) {
            return -1;
        }
        ret = System.identityHashCode(this) - System.identityHashCode(o);
        if (ret != 0) {
            return (int) ret;
        }
        // Jackpot! - different objects with same hashes
        throw new Error();
    }
```





`ChannelPipeLine`中的设计模式: 拦截过滤器.(类似于Unix管道命令)

Netty提供的`Channel`实现都是线程安全的，可以支持多个线程并发写一个`Channel`。


## Netty提供的传输实现
包括:
```java
NIO  : 基于选择器
Epoll: JNI驱动，支持linux上才有的特性,比NIO更快;
OIO :  阻塞流
Local: VM内部通过管道进行通信
Embedded: 测试时使用。
```

### NIO传输实现
{% img /images/2018-08/4.2.jpg 400 600 Channel接口 %}
实现上基于选择器，类似于一个注册表，用户注册各种事件的处理函数。
可能的状态变化包括:
op_accept: 接受新连接并创建`channel`时; 
op_connect: 建立一个连接时；
op_read: 可以读时；
op_write: 可以写时。


### Epoll传输实现
性能比NIO更高。代码更改:
```java
EpollEventLoopGroup
EpollServerSocketChannel.class
```

### 水平触发LT
> 只要有事件没消费完就一直提醒。(底层把没处理完的回调重新加入readylist)
 

### 边缘触发ET
> 有新事件才提醒。（底层省略了重新加入readylist那一步）


优缺点对比:

|    属性    | 水平触发(LT)   |  边缘触发(ET)  |  
| :--------:   | :----- | :-----: |  
|  开销 | 较大    |较小      |  
| 时效性| 时效性高  | 时效性较低     | 
| 数据安全| 无遗漏  | 需要小心编程，否则有遗漏风险     | 

单线程时: 水平触发时效性好一些；
高并发时: 边缘触发并发度好一些。


jdk实现: 水平触发
netty实现: 边缘触发

### OIO传输实现
Netty使用超时机制来让OIO和NIO的API统一。

### Local传输实现
用于在同一个JVM中运行的客户端和服务端之间的通信。 
服务端并不绑定物理网络地址，也不接受真正的网络流量。
可以一开始先使用`Local`传输实现，方便以后有需求的时候迁移到`NIO`,`Epoll`实现。

### Embedded传输实现
用途： 创建单元测试。

## 传输的用例
Netty4支持的网络协议:

|    传输实现    | TCP  |  UDP |  SCTP |  UDT |  
| :--------:   | :----- | :-----: |   :-----: |   :-----: |  
|  NIO |  √    |√       |  √       |  √       |  
|  Epoll | √     |√       |  X      |  X      |  
|  OIO | √     |√       |  √       |  √       |  
|  网络协议备注 |      |       |   串流控制传输协议，较冷门，目前主要应用在电信领域      |  基于UDP的可靠传输      |  
 