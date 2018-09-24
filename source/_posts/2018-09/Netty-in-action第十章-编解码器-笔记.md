---
title: Netty in action-第十章-编解码器-笔记
date: 2018-09-22 21:31:09
tags: 
- java
- Netty
categories:
- java
- Netty

---



编码器: encoder , 出站handler;
解码器: decoder,  入站handler。

用途包括:
`POP3`,`IMAP`,`SMTP`协议。（邮件服务器）

# 资源管理
编码器和解码器的消息被消费后，会自动调用`ReferenceCountUtil.release(message)`进行释放。
如果要阻止这种自动释放，可以显式调用`ReferenceCountUtil.retain(message)`保留消息。（后续再自己手动释放）


# 解码器
两种:
1.字节=>消息:`ByteToMessageDecoder`,`ReplayingDecoder`;
2.消息=>消息:`MessageToMessageDecoder`。(不需要检查`readableBytes`)

解码器包括:
```java
// 1.字节=>消息: （解码）
ByteToMessageDecoder: 抽象类;
LineBasedFrameDecoder: 行分割消息数据， 实际类。
HttpObjectDecoder: Http数据解析。抽象类。
// 2.消息=>消息: （格式转换）
MessageToMessageDecoder: 消息=>消息;
HttpObjectAggregator: Http数据转换。实际类。
```
相应的声明:
```java
public class LineBasedFrameDecoder extends ByteToMessageDecoder;

public abstract class HttpObjectDecoder extends ByteToMessageDecoder;

public class HttpObjectAggregator extends MessageAggregator<HttpObject, HttpMessage, HttpContent, FullHttpMessage>;

public abstract class MessageAggregator<I, S, C extends ByteBufHolder, O extends ByteBufHolder> extends MessageToMessageDecoder<I>;

```


## ByteToMessageDecoder(抽象类)
处理流程是字节=>消息，也就是`ByteBuf in`=>`List<Object>out`。
需要注意每次从`in`读之前，需要确认可读字节数量:`in.readableBytes()`。
具体代码:
```java
public class ToIntegerDecoder extends ByteToMessageDecoder {
    @Override
    public void decode(ChannelHandlerContext ctx
        ,ByteBuf in         // 1.输入
        ,List<Object> out)  // 2.输出
        throws Exception {
        //检查是否至少有 4 字节可读（一个 int 的字节长度）
        if (in.readableBytes() >= 4) {
            out.add(in.readInt());
        }
    }
}
```
`ByteToMessageDecoder`有俩api:
```java
decode:     必须实现，解析每条消息;
decodeLast: 可选，处理最后一条消息，默认是调用decode。
```

## ReplayingDecoder(抽象类)(尽量不用)
使用上比`ByteToMessageDecoder`省略`readableBytes`的调用，但是速度稍慢。
```java
public class ToIntegerDecoder2 extends ReplayingDecoder<Void> {
    @Override
    public void decode(ChannelHandlerContext ctx, ByteBuf in, 
    //传入的 ByteBuf 是 ReplayingDecoderByteBuf
        List<Object> out) throws Exception {
        //从入站 ByteBuf 中读取 一个 int，并将其添加到解码消息的 List 中
        out.add(in.readInt());
    }
}
```
尽可能使用`ByteToMessageDecoder`而不是`ReplaingDecoder`。
两个原因:
1. `ReplaingDecoder`比较慢;
2. 内部用的是`ReplayingDecoderByteBuf`，并没有支持所有`ByteBuf`的操作。(api不全)

## MessageToMessageDecoder(抽象类)
消息=>消息。
如果声明是`MessageToMessageDecoder<Integer>`,那么输入就是`Integer`类型。
具体签名是:
```java
public abstract class MessageToMessageDecoder<I>
extends ChannelInboundHandlerAdapter  
```
实际案例:
```java
public class IntegerToStringDecoder extends
    MessageToMessageDecoder<Integer> {
    @Override
    public void decode(ChannelHandlerContext ctx
        , Integer msg      // 1.输入
        , List<Object> out // 2.输出
) throws Exception {
        out.add(String.valueOf(msg));
    }
}
```
可以发现这种消息转换不需要检查`readableBytes`。(毕竟输入已经不是`byte`了)

## TooLongFrameException类
解码前，解码器会缓冲大量的数据。如果发现缓冲太多，可以抛出异常来报告这种情况:
```java
public class SafeByteToMessageDecoder extends ByteToMessageDecoder {
    private static final int MAX_FRAME_SIZE = 1024;
    @Override
    public void decode(ChannelHandlerContext ctx, ByteBuf in,
        List<Object> out) throws Exception {
            int readable = in.readableBytes();
            if (readable > MAX_FRAME_SIZE) {
                in.skipBytes(readable); // 一种可能的处理
                throw new TooLongFrameException("Frame too big!");// 抛出异常
        }
    }
}
```
抛异常的用意:
1.可能是上游生产太快,因此需要识别这种情况;
2.可能是下游消费太慢,也需要识别这种情况。

后续可以用`exceptionCaught`来捕获这个异常，可能的处理包括:
1.向生产端返回一个特殊的响应;
如HTTP:
`413` 错误 – 请求实体太大 (Request entity too large)
`414` 错误 – 请求 URI 过长 (Request URI too long)
2.关闭对应的连接。
3.其他处理方案。

# 编码器
两种: 
1.消息=>字节:MessageToByteEncoder;
2.消息=>消息:MessageToMessageEncoder。
// 主要是入站出站方向不同，不然和前面的消息转换差不多。


编码器包括:
```java
MessageToByteEncoder<I>: 消息=>字节
MessageToMessageEncoder: 消息=>消息
Websocket08FrameEncoder: 消息=>消息
ProtobufEncoder: 消息=>消息
```
具体代码:
```java
public class IntegerToStringEncoder
    extends MessageToMessageEncoder<Integer> {
    @Override
    public void encode(ChannelHandlerContext ctx
        , Integer msg       // 1.输入 
        , List<Object> out  // 2.输出
) throws Exception {
        out.add(String.valueOf(msg));
    }
}
```

解码器:`decode`,`decodeLast`
编码器:`encode`

# 编解码器（编码器+解码器复合）
编解码复合，也有两种:
1. 字节<=>消息: `ByteToMessageCodec`;
2. 消息<=>消息: `MessageToMessageCodec`.

可以发现就是多了一个`Codec`后缀。

## ByteToMessageCodec(抽象类)
`ByteToMessageCodec`的方法就是把编码器和解码器的api都加上:
```java
decode(ctx,ByteBuf in,List<Object>out)
decodeLast(ctx,ByteBuf in,List<Object>out)
encode(ctx,I msg,ByteBuf out)
```

## MessageToMessageCodec(抽象类)
这是一个参数化的类，声明如下；
```java
public abstract class MessageToMessageCodec<INBOUND_IN, OUTBOUND_IN> extends ChannelDuplexHandler
public class ChannelDuplexHandler extends ChannelInboundHandlerAdapter implements ChannelOutboundHandler
```
看到这里大致可以发现一个规律，这些编码器、解码器最重要的是定义输入数据的数据类型，而输出数据的数据类型一般都是`List<Object>`。
`MessageToMessageCodec`本质上是一个可以处理入站事件和出站事件的handler，因此需要定义入站和出站的数据类型:
```java
INBOUND_IN:  入站数据参数的数据类型
OUTBOUND_IN: 出站数据参数的数据类型
```

它的两个接口:
```java
protected abstract void encode(ChannelHandlerContext ctx
, OUTBOUND_IN msg   -- 输入
, List<Object> out) -- 输出，实际上一般会是INBOUND_IN类型
throws Exception;

protected abstract void decode(ChannelHandlerContext ctx
, INBOUND_IN msg    -- 输入
, List<Object> out) -- 输出，实际上一般会是OUTBOUND_IN类型
throws Exception;
```

## CombinedChannelDuplexHandler类
编解码器是从头写一个，实现双向转换。
`CombinedChannelDuplexHandler`是从已经写好的编码器和解码器，生成一个双向转换的封装类。
声明:
```java
public class CombinedChannelDuplexHandler<I extends ChannelInboundHandler
, O extends ChannelOutboundHandler>
        extends ChannelDuplexHandler {
```
传入两个类,I是inbound,O是outbound。具体样例如下:
```java
public class CombinedByteCharCodec extends
    CombinedChannelDuplexHandler<ByteToCharDecoder, CharToByteEncoder> {
    public CombinedByteCharCodec() {
        //将委托实例传递给父类
        super(new ByteToCharDecoder(), new CharToByteEncoder());
    }
}
```
传进来后调用父类默认写好的构造函数即可。


个人理解编解码器和DuplexHandler类都是高级用法，不一定实用。
个人偏好直接使用编码器、解码器就好了。

