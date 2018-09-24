---
title: Netty实战第三章-笔记
date: 2018-08-18 20:54:06
tags: 
- java
- Netty
categories:
- java
- Netty

---


# 外部接口
## Channel,EventLoop,ChannelFuture
{% img /images/2018-08/pic31.png 400 600 外部接口 %}
### Channel接口
封装`Socket`类，预定义的实现类包括:
```java
EmbeddedChannel
LocalServerChannel
NioDatagramChannel
NioSctpChannel
NioSocketChannel
...
```

### EventLoop
`EventLoop`: 只绑定一个专有线程。
`Channel`: 只注册于一个`EventLoop`；

`EventLoop`与线程: 1对1；
`EventLoop`与`Channel`: 1对多；
因此：线程与`Channel`：1对多。同一个`Channel`的IO操作一定由同一个线程完成。
{% img /images/2018-08/reactor.png 400 600 reactor模式 %}

### ChannelFuture接口
用于查询`Channel`的操作结果。

## ChannelHandler和ChannelPipeline

### ChannelHandler接口
用来处理事件的方法。
子接口：
`ChannelInboundHandler`:入站事件接口
`ChannelOutboundHandler`:出站事件接口


### ChannelPipeline接口
`ChannelHandler`接口的容器，就像一个`List<ChannelHandler>`。
是一个处理链。
在链上定义了入站和出站事件流。
{% img /images/2018-08/pic33.png 400 600 入站出站 %}
`ChannelHandler`安装过程:
```
1. 一个ChannelInitializer的实现注册到Bootstrap;
2. 调用initChannel,在pipeline中安装一组自定义的ChannelHandler;
    （2.1）ChannelHandler被安装时，分配一个ChannelHandlerContext给它，保存它与pipeline之间的绑定。
3. ChannelInitializer将自己从pipeline中移除。
```

### `ChannelHandlerContext`
可以用于获取底层的`Channel`。
两种写消息方式:
1. 写到`ChannelHandlerContext`中： 传递给`pipeline`中下一个`handler`;
2. 直接写`channel`：跳过后续的`handler`，直接到达尾端。


### 适配器
实际编写`ChannelHandler`的时候，一般不会直接用`ChannelInboundHandler`或者`ChannelOutboundHandler`接口，而会直接用预定义的实现类，然后进行扩展，也就是直接继承`extends`下列适配器类：
```java
ChannelHandlerAdapter
ChannelInboundHandlerAdapter
ChannelOutboundHandlerAdapter
ChannelDuplexHandler
```
好处是可以只重写自己关心的事件处理，其他的用默认的。

## ChannelHandler的子类型: 编码器/解码器
编码解码： 入站出站时候字节和对象的数据转换。

## 抽象类SimpleChannelInboundHandler
```java
SimpleChannelInboundHandler<I> extends ChannelInboundHandlerAdapter
```
要注意处理的数据类型。
由于上游可能有其他解码器，导致数据类型发生改变。