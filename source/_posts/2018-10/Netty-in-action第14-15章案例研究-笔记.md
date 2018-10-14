---
title: Netty-in-action第14-15章案例研究-笔记
date: 2018-10-14 12:00:00
tags: 
- java
- Netty
categories:
- java
- Netty

---

# 文件上传案例: Droplr
## 需求:
上传文件到S3,返回一个下载url。

数据流: 客户端=>服务端=>S3

## 原始方案:
1. 服务器接受上传,存成文件;
2. 服务器上传到S3;
3. 服务器回复url给客户端。

缺点: 
1. 每个上传开销大占用大量内存,导致并发低;
2. 上传完整个文件才开始上传S3,而瓶颈恰恰在于S3。
3. 有磁盘IO。

## 改进方案
流式上传,只在内存,不过磁盘。
1. 服务器接受上传，每块数据实时传输到S3;
2. 接受和上传做速度适配，保持低内存消耗，高并发；
3. 最后返回url给客户端。

并发达到10K。

### 要点
1. `IdelStateHandler`关闭不活跃的连接,回滚历史进度(速度控制失败时);
2. 并发达到上限时,返回503;
3. 需要保证`HttpChunk`的顺序(线程池)。
(注: 503: 服务不可用（服务器资源耗尽，拒绝服务）)

http-client库:
https://github.com/akka/akka-http
https://github.com/AsyncHttpClient/async-http-client

# 实时数据同步：FireBase（被谷歌收购）
## 需求1:
在各个用户和设备之间实时同步数据。
(服务器同步到各个客户端)

## 解决方案
长轮询+WebSocket.
先使用长轮询连接，当WebSocket可用时切换到WebSocket。

难点在于长轮询。

## 具体细节
轮询 : 客户端每隔N秒轮询一次;实时性不强,开销大（空转）;
长轮询 : 客户端询问一次服务器,然后等待服务器响应,收到响应以后才继续轮询。
在服务端没有回复客户端的期间,如果客户端此时想发送数据给服务端,它会先阻塞。
`此时，客户端手头的数据会堆积在缓冲区。`
换言之，限制是: `未完成请求数<=1`。(同一个客户端)


### 长轮询
优点: 没有新数据时,服务器可以不回应客户端,这样客户端就不会接着轮询,减少空转;
缺点: 客户端发送数据可能被阻塞。 

#### 改进
客户端: 限制改成`未完成请求数<=2`。(从1上调到2)
服务端: 如果当前有1个未完成请求`A`，此时又收到了同一个客户端的第二个请求`B`(一般第二个是发送数据的请求),会先对`A`进行空响应,然后处理请求`B`。

引入新问题: **消息的有序性;**
解决方案: 元数据加入消息序列号。

引入新问题: **连接断开检测**
解决方案: 
客户端: 超时重试(以区分于慢速网络)。
服务端: 超时判断为连接断开。

要点: Netty支持一个端口多个协议(HTTP,Websocket,长轮询,TCP)

## 需求2:
加密环境(`SSLHandler`)下，基于带宽计费。

### 方案:
1. 解密前统计字节数;
2. 解密后得到账户名,计入该账户的账单。

要点: 统计字节数提前到解密前，提高性能。

# app推送通知: Urban Airship案例
## 需求
实时推送通知

## 方案
1. app维护一条到后端服务的连接;
2. 借助第三方推送服务中转,服务器把消息传输给第三方平台，然后转交给app。

其中苹果的APNS推送服务的使用流程:
1. 生产者: 通过TCP+SSLv3连接到APNS服务器,使用X.509证书进行身份认证;
2. 生产者: 按APNS规定的格式，发送消息(二进制);
3. 生产者: 读取(消息id,错误码)或成功。(因为有消息id,这里可以异步)

其中消息格式是大端字节序，可以如下显式指定:
```java
ByteBuf buf = Unpooled.buffer(size).order(ByteOrder.BIG_ENDIAN);
```

初始化,设置允许重新协商密钥:
```java
final ChannelPipeline pipeline = channel.pipeline();
 final SslHandler handler = new SslHandler(clientEngine);
 handler.setEnableRenegotiation(true);// 重新协商
 pipeline.addLast("ssl", handler);
 pipeline.addLast("decoder", new ApnsResponseDecoder()); 
```

## 需要注意的经验
1. 运营商可能不允许TCP的keep-alive特性,会积极剔除空闲的TCP会话;
2. 移动运营商可能禁止UDP.

# 第15章 案例研究，第二部分
# 服务通信案例: Nifty和Swift(Facebook)
`Thrift`: facebook开发的跨语言rpc远程调用、服务通信的框架。
组件:
1. IDL: 定义通信的格式;
2. 协议；
3. 传输接口;
4. 编译器: 从IDL生成服务端和客户端的存根代码(不同语言);
5. 客户端和服务端实现。

## 场景:
由于`Thrift`是跨语言的远程调用。
其中C++版本基于`libevent`\ `Folly`开发，性能很高;
Java版本(`Nifty`)基于`Netty`开发，性能与C++版本不相上下。

**Nifty**：
> 基于`Netty`的`Thrift`java实现。

### 需求1： 按顺序响应
客户端可能会要求:
服务器端并行处理请求，但是返回响应必须是顺序的。

解决方案: 服务器端并行处理请求, 返回前排序响应。(缓冲处理好的请求结果)
开销: 缓冲响应的内存。（所以如果客户端不要求顺序响应，可以免除这部分开销）

Netty4的实现支持: `EventExecutor`
Netty3的实现支持: `OrderedMemoryAwareThreadPoolExcecutor`

**Swift**
> 用注解来定义模型，无效IDL文件和存根。
底层使用Nifty作为I/O引擎。
https://github.com/facebookarchive/swift
已经不再维护。
还在维护的类似开源项目是:
https://github.com/airlift/drift


## 超时处理
问题: 每个请求维护一个超时事件的话，代价很昂贵。

方案1: 超时集。
每个客户端维护一个计时器，或者每组相同超时间隔的请求，维护一个计时器。
每次超时结束以后，进行下一个超时计时器。
优点: 开销小;
缺点: 要求超时间隔长度一致。

方案2: 使用`Netty`的`HashedWheelTimer`工具类。（空间换时间）
算法来自: 
http://www.cs.columbia.edu/~nahum/w6998/papers/sosp87-timing-wheels.pdf
示例代码:
```java
HashedWheelTimer timer = new HashedWheelTimer(100, TimeUnit.MILLISECONDS, 16);
System.out.println(LocalTime.now());
timer.newTimeout((timeout) -> {
    System.out.println(LocalTime.now());
    System.out.println(timeout);
}, 5, TimeUnit.SECONDS);
//阻塞main线程
System.in.read();
```


# RPC框架案例: Finagle(Twitter)
前端api端点<=>`Finagle`<=>后端服务们(提供:用户信息、twitter、时间线)
(大部分是scala开发。原先是ruby on rails)

{% img /images/2018-10/15.6.png 400 600 Finagle %}
主要功能包括: SSL、打日志(统计)、负载均衡

### 负载均衡（故障管理）
客户端统计所有服务器的延迟、未完成请求数（负载），
每次选择最低负载的主机派发请求。

失败请求=>从列表中移除对应服务器=>后台不断尝试重连。
