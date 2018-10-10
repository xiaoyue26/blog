---
title: Netty-in-action-第十一章-SSL等预置实现-笔记
date: 2018-10-10 18:38:46
tags: 
- java
- Netty
categories:
- java
- Netty

---

这章主要讲内容:
1. SSL/TLS;
2. HTTP/HTTPS;
3. 空闲的连接和超时;
4. 基于分隔符和长度的协议（处理粘包，半包）;
5. 写大型数据。
6. 序列化

# SSL/TLS
安全协议：`SSL`/`TLS`
用例： `HTTPS`, `SMTPS`
实现： jdk实现(`javax.net.ssl`), `openssl`(性能更好)
Netty中的支持:
`SslHandler`
{% img /images/2018-10/11.1.png 400 600 sslhandler %}
`SslHandler`的声明:
```java
-- SslHandler:
public class SslHandler extends ByteToMessageDecoder implements ChannelOutboundHandler 
-- 其中的ByteToMessageDecoder:
public abstract class ByteToMessageDecoder extends ChannelInboundHandlerAdapter 
```
由声明看出，它是一个编解码器（入站事件和出站事件都处理）。
入站: 字节=>消息（解密）
出站: 消息=>字节（加密）
具体使用则和以前的编解码器都不同：
```java
public class SslChannelInitializer extends ChannelInitializer<Channel> {
    private final SslContext context;
    private final boolean startTls;

    public SslChannelInitializer(SslContext context,boolean startTls){
        this.context = context;
        this.startTls = startTls;
    }
    @Override
    protected void initChannel(Channel ch) throws Exception {
        SSLEngine engine = context.newEngine(ch.alloc());
        ch.pipeline().addFirst("ssl"
            , new SslHandler(engine, startTls));
    }
}
```
总结:
需要借用: SslContext

两个要注意的点:
1. 对于每个 `SslHandler` 实例，都使用 `Channel` 的 `ByteBufAllocator` 从 `SslContext` 获取一个新的 `SSLEngine`(`ch.alloc()`);
2. `startTls`: 如果设置为 true，第一个写入的消息将不会被加密;（客户端应该设置为 true）
3. 
https://github.com/devsunny/netty-ssl-example/blob/master/src/main/java/com/asksunny/ssl/StreamReader.java


# HTTP相关的handler
4个解码器、编码器：
```java
public class HttpPipelineInitializer extends ChannelInitializer<Channel> {

    private final boolean client;

    public HttpPipelineInitializer(boolean client) {
        this.client = client;
    }

    @Override
    protected void initChannel(Channel ch) throws Exception {
        ChannelPipeline pipeline = ch.pipeline();
        if (client) {
            pipeline.addLast("decoder", new HttpResponseDecoder());  //1
            pipeline.addLast("encoder", new HttpRequestEncoder());  //2
        } else {
            pipeline.addLast("decoder", new HttpRequestDecoder());  //3
            pipeline.addLast("encoder", new HttpResponseEncoder());  //4
        }
    }
}
```

## 消息聚合:
这回是编解码器`Codec`:
```java
public class HttpAggregatorInitializer extends ChannelInitializer<Channel> {

    private final boolean client;

    public HttpAggregatorInitializer(boolean client) {
        this.client = client;
    }

    @Override
    protected void initChannel(Channel ch) throws Exception {
        ChannelPipeline pipeline = ch.pipeline();
        if (client) {
            pipeline.addLast("codec", new HttpClientCodec());  //1
        } else {
            pipeline.addLast("codec", new HttpServerCodec());  //2
        }
        pipeline.addLast("aggegator", new HttpObjectAggregator(512 * 1024));  //3 512kb
    }
}
```


## HTTP 压缩
客户端加解压器，服务端加压缩器:
```java
public class HttpAggregatorInitializer extends ChannelInitializer<Channel> {

    private final boolean isClient;
    public HttpAggregatorInitializer(boolean isClient) {
        this.isClient = isClient;
    }
    @Override
    protected void initChannel(Channel ch) throws Exception {
        ChannelPipeline pipeline = ch.pipeline();
        if (isClient) {
            pipeline.addLast("codec", new HttpClientCodec()); //1
            pipeline.addLast("decompressor",new HttpContentDecompressor()); //2
        } else {
            pipeline.addLast("codec", new HttpServerCodec()); //3
            pipeline.addLast("compressor",new HttpContentCompressor()); //4
        }
    }
}
```

## HTTPS
`http`部分加上`sslHandler`就是`https`。不过本质上还是需要`SslContext`:
```java
public class HttpsCodecInitializer extends ChannelInitializer<Channel> {

    private final SslContext context;
    private final boolean client;

    public HttpsCodecInitializer(SslContext context, boolean client) {
        this.context = context;
        this.client = client;
    }

    @Override
    protected void initChannel(Channel ch) throws Exception {
        ChannelPipeline pipeline = ch.pipeline();
        SSLEngine engine = context.newEngine(ch.alloc());
        pipeline.addFirst("ssl", new SslHandler(engine));  //1

        if (client) {
            pipeline.addLast("codec", new HttpClientCodec());  //2
        } else {
            pipeline.addLast("codec", new HttpServerCodec());  //3
        }
    }
}
```

# WebSocket
`http`仅让客户端向服务端请求数据，服务端无法主动推数据给客户端。一种解决方案是让客户端轮询，另一种解决方案是`WebSocket`。
用`WebSocket`的话，底层是tcp双向连接，服务端可以主动发消息给客户端。
## WebSocket帧类型
三种数据帧:
```java
BinaryWebSocketFrame: 二进制;
TextWebSocketFrame: 文本;
ContunuationWebSocketFrame: 后续数据;
```
三种控制帧:
```java
PingWebSocketFrame: ping，对方会回pong;
PongWebSocketFrame: pong;
CloseWebSocketFrame: 关闭。
```

服务端示例:
```java
public class WebSocketServerInitializer extends ChannelInitializer<Channel> {
    @Override
    protected void initChannel(Channel ch) throws Exception {
        ch.pipeline().addLast(
            new HttpServerCodec(),
            //为握手提供聚合的 HttpRequest
            new HttpObjectAggregator(65536),
            //如果被请求的端点是"/websocket"，则处理该升级握手
            new WebSocketServerProtocolHandler("/websocket"),
            //TextFrameHandler 处理 TextWebSocketFrame
            new TextFrameHandler(),
            //BinaryFrameHandler 处理 BinaryWebSocketFrame
            new BinaryFrameHandler(),
            //ContinuationFrameHandler 处理 ContinuationWebSocketFrame
            new ContinuationFrameHandler());
    }

    public static final class TextFrameHandler extends
        SimpleChannelInboundHandler<TextWebSocketFrame> {
        @Override
        public void channelRead0(ChannelHandlerContext ctx,
            TextWebSocketFrame msg) throws Exception {
            // Handle text frame
        }
    }

    public static final class BinaryFrameHandler extends
        SimpleChannelInboundHandler<BinaryWebSocketFrame> {
        @Override
        public void channelRead0(ChannelHandlerContext ctx,
            BinaryWebSocketFrame msg) throws Exception {
            // Handle binary frame
        }
    }

    public static final class ContinuationFrameHandler extends
        SimpleChannelInboundHandler<ContinuationWebSocketFrame> {
        @Override
        public void channelRead0(ChannelHandlerContext ctx,
            ContinuationWebSocketFrame msg) throws Exception {
            // Handle continuation frame
        }
    }
}
```

# 空闲事件、超时事件
`WebSocket`协议中多了几种事件: 

| 触发时机        | 事件   |  处理方法  |  预置handler
| :--------:   | :----- | :-----: | :-----: 
| 空闲时间超过配置 | IdleStateEvent     | userEventTriggered()| IdleStateHandler
| 指定时间间隔内没有收到入站数据| ReadTimeoutException  | exceptionCaught()| ReadTimeoutHandler
|指定时间间隔内没有出站数据 | WriteTimeoutException   | exceptionCaught()| WriteTimeoutHandler

具体使用方法:
1. 注册预置的handler,截获对应的事件;(`IdleStateHandler`,`ReadTimeoutHandler`,`WriteTimeoutHandler`)
2. 实现一个自定义handler注册到pipeline,处理对应的事件。 


空闲事件示例:
1. 注册`IdleStateHandler`，负责截获空闲事件，它会调用`fireUserEventTriggered`方法,触发`userEvent`事件;
2. 实现自定义`handler`,处理`userEvent`:一种可能的处理逻辑是进行心跳检测，检测到是空闲事件就发送心跳,发送失败就关闭连接; 如果不是空闲事件,则抛出去,让下一级处理。
```java
public class IdleStateHandlerInitializer extends ChannelInitializer<Channel>
    {
    @Override
    protected void initChannel(Channel ch) throws Exception {
        ChannelPipeline pipeline = ch.pipeline();
        pipeline.addLast(
//(1)IdleStateHandler 将在被触发时发送一个IdleStateEvent事件:
                new IdleStateHandler(0, 0, 60, TimeUnit.SECONDS));
//(2)将一个HeartbeatHandler添加到ChannelPipeline中:
        pipeline.addLast(new HeartbeatHandler());
    }

//(3)实现userEventTriggered()方法以发送心跳消息:
    public static final class HeartbeatHandler
        extends ChannelInboundHandlerAdapter {
        //发送到远程节点的心跳消息
        private static final ByteBuf HEARTBEAT_SEQUENCE =
                Unpooled.unreleasableBuffer(Unpooled.copiedBuffer(
                "HEARTBEAT", CharsetUtil.ISO_8859_1));
        @Override
        public void userEventTriggered(ChannelHandlerContext ctx,
            Object evt) throws Exception {
//(4)发送心跳消息，并在发送失败时关闭该连接
            if (evt instanceof IdleStateEvent) {
                ctx.writeAndFlush(HEARTBEAT_SEQUENCE.duplicate())
                     .addListener(
                         ChannelFutureListener.CLOSE_ON_FAILURE);
            } else {
                //若不是IdleStateEvent事件，所以将它传递给下一个ChannelInboundHandler
                super.userEventTriggered(ctx, evt);
            }
        }
    }
}
```

# 工具：解决粘包和半包(数据帧的划分问题)
`netty`主要是字节流层传输，并不关心应用层对数据的划分（并不关心帧是如何划分的）。
但是`netty`提供了很多帮助分隔帧的工具类，来解决粘包和半包的问题。
数据帧的划分问题一般有三种解决方案:
1. 定长帧;
2. 指定分隔符;
3. head-body结构，header中规定body长度。(`HTTP`)比较灵活，比较常见。

## 指定分隔符
相关工具类: `DelimitedBasedFrameDecoder`,`LineBasedFrameDecoder`

## 定长帧
相关工具类: `FixedLengthFrameDecoder`

## head-body结构
相关工具类: `LengthFieldBasedFrameDecoder`


# 高级特性: 写大文件(或大数据)
两种实现:
1. 直接写文件: `FileRegion`;
2. 借助预置实现:`ChunkedWriteHandler`。

## FileReion
直接在`channel`中写入`FileRegion`即可:(还可以用`ChannelProgressivePromise`来获取传输进度)
```java
FileInputStream in = new FileInputStream(file);
//以该文件的完整长度创建一个新的 DefaultFileRegion
FileRegion region = new DefaultFileRegion(
        in.getChannel(), 0, file.length());
//发送该 DefaultFileRegion，并注册一个 ChannelFutureListener
channel.writeAndFlush(region).addListener(
    new ChannelFutureListener() {
    @Override
    public void operationComplete(ChannelFuture future)
       throws Exception {
       if (!future.isSuccess()) {
           //处理失败
           Throwable cause = future.cause();
           // Do something
       }
    }
});
```

## ChunkedWriterHandler
数据流是:
数据源=>`ChunkedInput`=>自定义的StreamHandler=>`ChunkedWriteHandler`=>出站

其中`ChunkedInput`有4种实现:

| 实现名称        | 数据源   |   备注
| :--------:   | :----- |   :-----: | 
| ChunkedFile | 文件   |   当平台不支持零拷贝，或需要转换数据时使用
| ChunkedNioFile|  文件| 使用FileChannel
|ChunkedStream |   InputStream | 
|ChunkedNioStream |  ReadableByteChannel |

示例代码:
```java
@Override
protected void initChannel(Channel ch) throws Exception {
    ChannelPipeline pipeline = ch.pipeline();
    //将 SslHandler 添加到 ChannelPipeline 中
    pipeline.addLast(new SslHandler(sslCtx.newEngine(ch.alloc())));
    //添加 ChunkedWriteHandler 以处理作为 ChunkedInput 传入的数据
    pipeline.addLast(new ChunkedWriteHandler());
    //一旦连接建立，WriteStreamHandler 就开始写文件数据
    pipeline.addLast(new WriteStreamHandler());
}

public final class WriteStreamHandler
extends ChannelInboundHandlerAdapter {
@Override
//当连接建立时，channelActive() 方法将使用 ChunkedInput 写文件数据
    public void channelActive(ChannelHandlerContext ctx)
        throws Exception {
        super.channelActive(ctx);
        ctx.writeAndFlush(
        new ChunkedStream(new FileInputStream(file))); // ChunkedStream => ChunkedInput
    }
}
```

# 序列化数据
这里介绍3种方法:
1. JDK的`ObjectOutputStream`;
2. JBoss marshalling;
3. Protocol buffers.
## JDK序列化
只要实现了`Serializable`接口的对象，就可以使用`ObjectOutputStream`。
示例代码:
```java
FileOutputStream fos = new FileOutputStream("t.tmp");
ObjectOutputStream oos = new ObjectOutputStream(fos);
oos.writeObject(new Date());
oos.close();
```

Netty提供的速度优化:
`ObjectInputStream` =>`ObjectDecoder`
`ObjectOutputStream`=>`ObjectEncoder`

## JBoss Marshalling序列化
比JDK序列化快3倍。
`MarshallingDecoder`
`MarshallingEncoder`

示例代码:
```java
public class MarshallingInitializer extends ChannelInitializer<Channel> {
    private final MarshallerProvider marshallerProvider;
    private final UnmarshallerProvider unmarshallerProvider;

    public MarshallingInitializer(
            UnmarshallerProvider unmarshallerProvider,
            MarshallerProvider marshallerProvider) {
        this.marshallerProvider = marshallerProvider;
        this.unmarshallerProvider = unmarshallerProvider;
    }

    @Override
    protected void initChannel(Channel channel) throws Exception {
        ChannelPipeline pipeline = channel.pipeline();
        //添加 MarshallingDecoder 以将 ByteBuf 转换为 POJO
        pipeline.addLast(new MarshallingDecoder(unmarshallerProvider));
        //添加 MarshallingEncoder 以将POJO 转换为 ByteBuf
        pipeline.addLast(new MarshallingEncoder(marshallerProvider));
        //添加 ObjectHandler，以处理普通的实现了Serializable 接口的 POJO
        pipeline.addLast(new ObjectHandler());
    }

    public static final class ObjectHandler
        extends SimpleChannelInboundHandler<Serializable> {
        @Override
        public void channelRead0(
            ChannelHandlerContext channelHandlerContext,
            Serializable serializable) throws Exception {
            // Do something
        }
    }
}
```

其中provider的创建代码:
```java
 public static MarshallingDecoder buildMarshallingDecoder() {
        //首先通过Marshalling工具类的getProvidedMarshallerFactory静态方法获取MarshallerFactory实例
        //参数“serial”表示创建的是Java序列化工厂对象，它由jboss-marshalling-serial-1.3.0.CR9.jar提供。
        final MarshallerFactory marshallerFactory = Marshalling.getProvidedMarshallerFactory("serial");
        //创建了MarshallingConfiguration对象
        final MarshallingConfiguration configuration = new MarshallingConfiguration();
        //将它的版本号设置为5
        configuration.setVersion(5);
        //然后根据MarshallerFactory和MarshallingConfiguration创建UnmarshallerProvider实例
        UnmarshallerProvider provider = new DefaultUnmarshallerProvider(marshallerFactory, configuration);
        //最后通过构造函数创建Netty的MarshallingDecoder对象
        //它有两个参数，分别是UnmarshallerProvider和单个消息序列化后的最大长度。
        MarshallingDecoder decoder = new MarshallingDecoder(provider, 1024);
        return decoder;
    }
```


## Protocol Buffer序列化
google的序列化方案。
主要是4个类:
```java
// 解码:
ProtobufVarint32FrameDecoder: bytes=>msg; 解析出头部的长度字段,以正确划分帧;
ProtobufDecoder: msg=>msg;
// 编码:
ProtobufVarint32LengthFieldPrepender: msg=>bytes; 头部添加长度字段.
ProtobufEncoder: msg=>msg. 
```


示例代码:
服务端:
```java
.childHandler(new ChannelInitializer<SocketChannel>() {
    @Override
    protected void initChannel(SocketChannel ch) throws Exception {
        ch.pipeline()
                .addLast(new ProtobufVarint32FrameDecoder())
                .addLast(new ProtobufDecoder(
                        ProtoObject.Req.getDefaultInstance()))
                .addLast(new ProtobufVarint32LengthFieldPrepender())
                .addLast(new ProtobufEncoder())
                .addLast(new ServerHandler());
    }
})
```
客户端:
```java
.childHandler(new ChannelInitializer<SocketChannel>() {
    @Override
    protected void initChannel(SocketChannel ch) throws Exception {
        ch.pipeline()
                .addLast(new ProtobufVarint32FrameDecoder())
                .addLast(new ProtobufDecoder(
                        ProtoObject.Resp.getDefaultInstance()))
                .addLast(new ProtobufVarint32LengthFieldPrepender())
                .addLast(new ProtobufEncoder())
                .addLast(new ClientHandler());
    }
})
```
`ProtobufDecoder`实际上可以接受`MessageLite`或者`Builder`。
`Message`是`MessageLite`的子接口，因此可以用`Message`代替`MessageLite`。(基类指针存放子类对象)
```java
public interface Message extends MessageLite, MessageOrBuilder {...}
```


