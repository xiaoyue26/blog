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
userEventTriggered: 调用ChannelInboundHandler.fireUserEventTriggered()时触发。用于用户自定义事件。

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

# 入站事件和出站事件的区别
再次强调一件事: 
> 出站事件基本比入站事件都多了1个`ChannelPromise`参数。

本质上也就是出站事件要多一个通知机制: 
`ChannelPromise`,`ChannelFuture`与`ChannelFutureListener`。

- 那么为什么两者要这样区别呢? 
根本原因是出站中有`write`,`flush`这样的`io`操作，比较费时而且依赖于复杂因素，需要设计成异步的。
而入站事件基本都是在自己的内存里搞定，同步就够用了。

理解了这一点，我们就能心平气和地接受出站事件的通知机制了。

## 通知机制

> 1. 每个出站操作返回一个`ChannelFuture`，注册到它的`ChannelFutureListener`将在操作完成的时候被通知成功还是失败；
2. 出站操作传入一个`ChannelPromise`,可以进行立即通知(更改状态): `setSuccess`/`setFailure`。

注册`ChannelFutureListener`的两种姿势:
1. 对channel进行操作,获取`ChannelFuture`,然后注册`Listener`; // 可以用于某一次写的定制化操作;
2. 出站事件中,在传入的`ChannelPromise`上注册`Listener`。 // 应用于某类型的所有操作。

相应的代码如下:
```java
// 方法1:
io.netty.channel.ChannelFuture future = channel.write(someMessage);
        future.addListener(new ChannelFutureListener() {
            @Override
            public void operationComplete(io.netty.channel.ChannelFuture f) {
                if (!f.isSuccess()) {
                    f.cause().printStackTrace();
                    f.channel().close();
                }
            }
        });
// 方法2:
public class OutboundExceptionHandler extends ChannelOutboundHandlerAdapter {
    @Override
    public void write(ChannelHandlerContext ctx, Object msg,
        ChannelPromise promise) {
        promise.addListener(new ChannelFutureListener() {
            @Override
            public void operationComplete(ChannelFuture f) {
                if (!f.isSuccess()) {
                    f.cause().printStackTrace();
                    f.channel().close();
                }
            }
        });
    }
}
```

## ChannelHandler适配器
netty提供了`handler`的基本实现:`ChannelInboundHandlerAdapter`和`ChannelOutboundHandlerAdapter。`（入站和出站）

# ChannelPipeline接口

> 1个Channel <=> 对应1个固定的ChannelPipeline

在一个`ChannelHandler`中如何访问`pipeline`?:
> 通过context获取到pipeline即可。

### 传播事件
测试下一个`ChannelHandler`的类型是否与方向一致。

### pipeline编排
```java
    public static void modifyPipeline() {
        ChannelPipeline pipeline = CHANNEL_PIPELINE_FROM_SOMEWHERE; // get reference to pipeline;
        //创建一个 FirstHandler 的实例
        FirstHandler firstHandler = new FirstHandler();
        //将该实例作为"handler1"添加到ChannelPipeline 中
        pipeline.addLast("handler1", firstHandler);
        //将一个 SecondHandler的实例作为"handler2"添加到 ChannelPipeline的第一个槽中。这意味着它将被放置在已有的"handler1"之前
        pipeline.addFirst("handler2", new SecondHandler());
        //将一个 ThirdHandler 的实例作为"handler3"添加到 ChannelPipeline 的最后一个槽中
        pipeline.addLast("handler3", new ThirdHandler());
        //...
        //通过名称移除"handler3"
        pipeline.remove("handler3");
        //通过引用移除FirstHandler（它是唯一的，所以不需要它的名称）
        pipeline.remove(firstHandler);
        //将 SecondHandler("handler2")替换为 FourthHandler:"handler4"
        pipeline.replace("handler2", "handler4", new FourthHandler());
    }
```

## pipeline的事件API
主要用于触发下一个`handler`的事件,触发下一个入站事件一般带个前缀`fire`，触发出站事件则没有这个前缀。
例如:
`fireChannelRegistered`: 触发pipeline中下一个channelInboundHandler的`channelRegistered`事件。(注意是`Inbound`)
`connect`: 将channel连接到一个远程地址，将调用下一个`channelOutboundHandler`的connect方法。(注意是`outbound`)


# ChannelHandlerContext接口
`ChannelHandlerContext`记录`channelHandler`和`channel`的联系，类似于一个弱实体。
它也有很多事件API，含义与其他类的不同，是基于当前上下文的，也就是说:
从当前关联的`ChannelHandler`开始，传播给**下一个**。

`ChannelHandlerContext`部分API:
// Channel相关:
`alloc`: 返回`Channel`的`ByteBufAllocator`;
`executor`: 返回调度事件的`EventExecutor`;
// handler相关:
`fireChannelRead`: 触发下一个`InboundHanlder`的`ChannelRead`方法;(入站)
`write`: 通过当前实例写入消息,并经过pipeline。

```java
public interface ChannelHandlerContext extends AttributeMap, ChannelInboundInvoker, ChannelOutboundInvoker 
```
如上所示,这个接口其实继承了inbound和outbound,所以出站方法也有。

{% img /images/2018-09/6.5.jpg 400 600 事件传递 %}
综上所述，如果要从某一个`ChannelHandler`A开始传递事件，要先获得它的上一个handler的context。如上图所示，调用`pipeline`或者`channel`上的事件的话，事件就会从1号位置开始流动,调用`channelContext`上的事件则会从2,3号位置(也就是下一个)开始流动。

优势:
1. 减少事件传播开销;
2. 避开一些`handler`的处理。

上述API可能的具体用途:
1. 动态切换协议. 
2. 其他用途(暂时不知道还有啥)

## Sharable
可以将一个`ChannelHandler`绑定到多个`pipeline`(此时会产生多个`ChannelHandlerContext`)。这样做的场景: 比如需要收集跨越多个`Channel`的统计信息时。

加上`@Sharable`注解的`ChannelHandler`(语法上)可以绑定到多个`pipeline`上，但程序员需要注意解决线程安全的问题。// 要么无状态不可变，要么加锁，要么CAS，要么threadlocal。

# 异常处理
参考前文中出站事件的通知机制，因此出站事件中的异常也是封装在`ChannelFuture`中的，而不是像入站事件用`exceptionCaught`。
// 换言之, `ChannelOutboundHandler`没有`exceptionCaught`API。

入站事件异常:// 消费完异常才不会向尾端传播
```java
@Override
    public void exceptionCaught(ChannelHandlerContext ctx,
        Throwable cause) {
        cause.printStackTrace();
        ctx.close();
    }
```

出站事件异常:
```java
@Override
public void operationComplete(io.netty.channel.ChannelFuture f) {
    if (!f.isSuccess()) {
        f.cause().printStackTrace();
        f.channel().close();
    }
}
```