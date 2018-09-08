---
title: Netty in action第六章-ChannelHandler和pipeline-笔记
date: 2018-09-08 20:16:07
tags: 
- java
- netty
categories:
- java
- netty

---

# ChannelHandler和ChannelPipeline

## Channel生命周期 / 状态自动机
`ChannelUnregisterd`=>`ChannelRegistered` => `ChannelActive` => `ChannelInactive` => `ChannelUnregisterd`
每一个状态转化都会产生相应事件。

> ChannelUnregistered：刚创建;
ChannelRegistered: (创建以后),已注册到EventLoop;
ChannelActive: 已经连接到远程节点;
ChannelInactive: 没有连接到远程节点。

## ChannelHeadler 生命周期
> handlerAdded: ChannelHeadler添加到pipeline时调用;
handlerRemoved: 移除时;
exceptionCaught: 发生错误时。

## ChannelInboundHandler接口 (入站事件)
**省略别的常见事件:**
> ChannelWritabilityChanged: 可写状态发生改变事件
userEventTriggered: 调用ChannelInboundHandler.fireUserEventTriggered()时调用。

### 可写状态与高低水位:
`high watermark`机制: 写太快时达到高水位线时，转变为不可写; 
// is_full()是根据当前是否大于等于high water mark来判断，如果full会wait。
`low watermark`机制: 达到低水位线时，转变为可写。

// 其他地方`low watermark`的含义: 设定最小时间戳，低于低水位线的数据不再接收。

### 高低水位设置:
```java
Channel.config.setWriteHighWaterMark();
Channel.config.setWriteLowWaterMark();
```

`ChannelConfig`默认的水位配置为低水位32K，高水位64K。


### 资源释放
第五章里提到了`bufferBuf`的释放问题:
`pipeline`里最后一个`Handler`要负责释放收到的数据:
```java
bufferBuf.release();
```

落实到入站事件中, 如果重写了`ChannelRead()`事件,这个方法需要负责释放池化的`ByteBuf`:
```java
@Override
public void channelRead(ChannelHandlerContext ctx, Object msg) {
    //通过调用 ReferenceCountUtil.release()方法释放资源
    ReferenceCountUtil.release(msg);
}
// 底层调的是这个:
public static boolean release(Object msg) {
        if (msg instanceof ReferenceCounted) {
            return ((ReferenceCounted) msg).release();
        }
        return false;
}
```

## SimpleChannelInboundHandler的自动释放
如果不想每次在`ChannelRead()`方法里释放消息,可以直接使用`SimpleChannelInboundHandler`,它会自动释放收到的消息。
相对的，由于有了自动释放,后续就无法再访问到了,因此使用`SimpleChannelInboundHandler`的时候消息引用会失效。

小结:
> 1. 不使用SimpleChannelInboundHandler: 记得在一个ChannelRead释放消息数据;
2. 使用SimpleChannelInboundHandler: 注意会被自动释放，引用会失效。

## ChannelOutboundHandler接口(出站事件)
类似的，出站事件也有很多，省略一些常见，列出几个特别的:
```
flush: 将数据冲刷到远程节点时被调用;
write: 将数据写到远程节点时被调用;
// 一旦ByteBuf 被写入到远程端， 它立即自动地放回原来的buffer池中.
```

与入站事件相对的,需要在`write`方法中释放消息:
```java
@Override
public void write(ChannelHandlerContext ctx,
    Object msg, ChannelPromise promise) {
    ReferenceCountUtil.release(msg);
    //通知 ChannelPromise数据已经被处理了
    promise.setSuccess();
}
```

注意到`write`方法比`channelRead`多一个`ChannelPromise`参数:
```java
ChannelPromise(子接口) -> ChannelFuture(父接口)
```

## ChannelPromise与ChannelFuture
`设计模式`:
实际上出站事件基本都多了这个`ChannelPromise`参数。
为了避免程序员写`bug`，netty4用`ChannelPromise`接口来更改任务完成状态,
而在那些只需要读/查询的场景，返回`ChannelFuture`接口。

此外,`ChannelFuture`中比jdk的普通`Future`多了一些信息,状态有4种:
```java
Uncompleted => success/fail/cancelled
```
每种状态的判定:(其实就是字面上的意思,猜也知道)

| 状态        | 判定条件  |   
| :--------:   | :----- | 
| Uncompleted |  isDone():false,isSuccess():false,isCancelled():false,cause():null     | 
| success| isDone():ture,isSuccess():ture  | 
|fail |isDone():ture,isSuccess():false,cause():non-null   |   
|cancelled |isDone():ture,isCancelled():true   |   
