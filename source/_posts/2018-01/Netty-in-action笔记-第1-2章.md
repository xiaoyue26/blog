---
title: Netty in action笔记-第1-2章
date: 2018-01-13 22:16:24
tags: 
- java 
- NIO 
- Netty
categories:
- java
- Netty
---


Hadoop中用了Netty3,所以得看看这一块.
中文版的代码链接:
https://github.com/ReactivePlatform/netty-in-action-cn


# 第一章 Netty介绍
这一章主要介绍了一下Netty.
Netty封装了Java NIO中的一些复杂的细节和坑.

## 1.1 为什么使用Netty
1. Netty提供了高层次抽象来简化TCP/UDP服务器的编程.
用Netty可以实现FTP,SMTP,HTTP,WebSocket,SPDY.
2. Netty社区很活跃.

### 1.1.2 Netty框架的组成
{% img /images/netty-chapter1.png 400 600 Netty框架 %}

## 1.2 异步设计
Netty中主要使用了回调+Future. 
### 1.2.1 回调
异步处理的一种技术是回调. 就是在一些关心的事件上注册回调函数.
// 个人理解,回调应该是同步非阻塞. 
// 同步: 客户端等待注册完成;
// 非阻塞: 服务端只是注册事件,返回得很快.

###  1.2.2 Future
`java.util.concurrent`包中附带的Future接口.使用`Executor`异步执行.
每传递一个Runnable对象到`ExecutorService.submit()`方法就会得到一个回调的Future,能使用它检测是否执行完成.
// 个人理解,Future其实是异步阻塞.
// 异步: 客户端不等待服务端执行结束,拿到Future后,需要自己轮询结果.
// 阻塞: 服务端不会主动回调,只是提供一个查询接口.

### 1.4 Netty相比NIO优点
1. 兼容性和跨平台性进一步提高;
2. 扩展ByteBuffer. 
`ByteBuffer`允许包装一个`byte[]`来获得一个实例,可以尽量减少内存拷贝.
3. 消除NIO的内存泄漏(jdk1.7以上)
NIO对于缓存区的聚合和分散操作可能造成内存泄漏.
- 分散(Scatter)
> 将`ScatteringByteBuffer`中的数据分散到多个`ByteBuffer`中.

- 聚合(Gather)
> 将多个`ByteBuffer`的数据聚合到`GatheringByteChannel`中.

4. 解决epoll缺陷导致的100%cpu问题.

# 第二章 Netty核心概念,简单示例
这章用的是Netty4.(但愿和Netty3同理)
上来先整了几段代码,而没有先说概念. 
代码如下:
https://github.com/xiaoyue26/netty-in-action-cn/blob/ChineseVersion/chapter2/Server/src/main/java/nia/chapter2/echoserver/EchoServer.java

可以看出服务端和客户端代码很类似,大致套路是:
1. 创建一个EventLoopGroup;//类似于召集一群干活的(线程池)
2. 创建一个Bootstrap(ServerBootstrap);// 类似于管家/控制面板
3. Bootstrap配置上eventGroup,channel用的类,端口地址,处理链.
4. 绑定到端口开始工作. 

值得注意的是,所有涉及到Future的方法都是异步的,可以通过主动调用`sync`方法来进行同步等待.(当然也可以轮询)

- 设计思想
1. 在Bootstrap上使用Future;
2. 在处理链上使用回调.

- 具体细节
1. `ChannelInboundHandlerAdapter`:
处理完消息后需要释放资源;(`ByteBuf.release()`)
2. `SimpleChannelInboundHandler`:
完成channelRead0后自动释放消息.