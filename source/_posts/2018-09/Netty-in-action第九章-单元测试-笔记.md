---
title: Netty in action第九章-单元测试-笔记
date: 2018-09-22 21:26:52
tags: 
- java
- netty
categories:
- java
- netty

---


# EmbeddedChannel(测试Handler)
实际应用中,我们大概率需要编写各种`inboundHandler`,`outboundHandler`，这个时候可以使用`EmbeddedChannel`来测试处理逻辑。
(测`Handler`用的)
{% img /images/2018-09/9.1.png 400 600 单元测试 %}
| api       | 作用   |   
| :--------:   | :----- |  
| writeInbound | 往测试channel里写数据.测试入站事件。如果这条数据能够通过所有`InboundHandler`，则返回`true`      | 
| writeOutbound| 往测试channel里写数据.测试出站事件。如果这条数据能够通过所有`OutboundHandler`，则返回`true`      | 
| readInbound | 读入站消息，无则返回`null`   |  
| readOutbound| 读出站消息，无则返回`null` |  
| finish   | 关闭写。还可以读。     |  




示例代码:
```java
@Test
public void testFramesDecoded() {
    //创建一个 ByteBuf，并存储 9 字节
    ByteBuf buf = Unpooled.buffer();
    for (int i = 0; i < 9; i++) {
        buf.writeByte(i);
    }
    ByteBuf input = buf.duplicate(); // 1
    EmbeddedChannel channel = new EmbeddedChannel(
        new FixedLengthFrameDecoder(3));
     
    assertTrue(channel.writeInbound(input.retain())); // 2. 数据量够,能够传到末端
    assertTrue(channel.finish()); // 3. 还有可读数据,返回true
   
    // 读3帧(3个入站消息):
    ByteBuf read = (ByteBuf) channel.readInbound();
    assertEquals(buf.readSlice(3), read); // 4
    read.release(); // 5

    read = (ByteBuf) channel.readInbound();
    assertEquals(buf.readSlice(3), read);
    read.release();

    read = (ByteBuf) channel.readInbound();
    assertEquals(buf.readSlice(3), read);
    read.release();

    assertNull(channel.readInbound());
    buf.release();
}
```

可以结合`JUnit`编写单元测试。
上述代码中标了几个要注意的地方:

- 1.duplicate和copy:
> duplicate: 浅拷贝buf,生成input。数据是共享的，writerIndex和readerIndex是隔离的。
copy: 深拷贝，数据和index都是隔离的。

- 2.retain和release:
> retain: 引用计数器+1;
release: 引用计数器-1。
因为进行了浅拷贝，把引用传递给方法之前必须调一次`retain`。
(如果只是和父ByteBuf同生命周期范围,不传递,可以不调。大部分时候还是调一下为好。)
最后使用结束记得要release。

- 3.channel.finish():
> channel.finish():
标记这个channel已经不能再写入数据了。(但是可以读)

- 4.buf.readSlice(3);
> 返回值类似于buf.slice(3), 但是有一个read前缀,说明这个方法调用结束后会增加readerIndex。(消费行为)
作用是读出3个字节来。

- 5.release: 
> 使用结束记得release。

 
上述代码是测试入站事件,所以样例用的是`inbound`和`decoder`；
如果要测试出站事件,只要类似改成`outbound`和`encoder`即可。


## 异常测试
除了出站入站事件,还可以测试异常捕捉(`exceptionCaught`)。

- 如果写了`exceptionCaught`，期望的结果是测试中不能捕捉到异常;
- 反之如果没写,期望的结果是测试中能捕捉到异常。


前者:期望不能捕捉到(被`exceptionCaught`处理了)
```java
try {
        channel.writeInbound(input.readBytes(4));
    } catch (TooLongFrameException e) {
    Assert.fail();// 期望不能抵达这里
    }
}
```


后者:期望能捕捉到:
```java
try {
            channel.writeInbound(input.readBytes(4));
            Assert.fail();// 期望不能抵达这里。
        } catch (TooLongFrameException e) {
        }
}
```

主要还是逻辑上的控制流测试。