---
title: Netty in action第八章-引导-笔记
date: 2018-09-17 10:52:07
tags: 
- java
- netty
categories:
- java
- netty

---

# AbstractBootstrap

引导: Bootstrapp
可以理解成一个对外的接口，可以把前面几章提到的内部组件封装起来，对外提供服务使用。

{% img /images/2018-09/8.1.png 400 600 bootstrap %}
服务端: `ServerBootstrap`，一个父Channel创建多个子Channel;
客户端: `Bootstrap`,一个普通Channel用于所有网络通信。


`Bootstrap`和`ServerBootstrap`都继承自`AbstractBootstrap`,具体声明如下:
```java
public abstract class AbstractBootstrap<B extends AbstractBootstrap<B, C>, C extends Channel> implements Cloneable {}

public class Bootstrap extends AbstractBootstrap<Bootstrap, Channel> {}

public class ServerBootstrap extends AbstractBootstrap<ServerBootstrap, ServerChannel> {}
```
1. 为什么语法看起来有点复杂呢?
- 其实是为了配置类的接口能返回自身的类型的引用。

2. 为什么需要这样呢? (配置类的接口能返回自身的类型的引用) 
- 根本目的是为了配置的时候实现流式语法糖,类似于builder的设计模式:
```java
bootstrap.group(group)
        .channel(NioSocketChannel.class)
        .handler(new SimpleChannelInboundHandler<ByteBuf>() {
            @Override
            protected void channelRead0(
                ChannelHandlerContext channelHandlerContext,
                ByteBuf byteBuf) throws Exception {
                System.out.println("Received data");
            }
            });
```


不理解的话，具体看接口就理解了。
两者继承自`AbstractBootstrap`的接口有两类。

第一类, 返回`B`的,参照上面的声明，这里的`B`类型其实就是自身的类型,具体映射如下:
`Bootstrap`: `B`=`Bootstrap`,`C`=`Channel`;
`ServerBootstrap`:`B`=`ServerBootstrap`,`C`=`ServerChannel`:
记住这个映射关系,就能看懂下面的源码了:(Netty4)
```java
public B channel(Class<? extends C> channelClass) {// 设置Channel
        if (channelClass == null) {
            throw new NullPointerException("channelClass");
        }
        return channelFactory(new ReflectiveChannelFactory<C>(channelClass));
    }
public B group(EventLoopGroup group);// 设置EventLoop实现
public B localAddress(SocketAddress localAddress) {// 设置本地地址
        this.localAddress = localAddress;
        return (B) this;
    }
public B localAddress(String inetHost, int inetPort) {// 设置本地地址
        return localAddress(SocketUtils.socketAddress(inetHost, inetPort));
    }
public <T> B option(ChannelOption<T> option, T value) { // 设置全局ChannelOption
        if (option == null) {
            throw new NullPointerException("option");
        }
        if (value == null) {
            synchronized (options) {
                options.remove(option);
            }
        } else {
            synchronized (options) {
                options.put(option, value);
            }
        }
        return (B) this;
  }
public <T> B attr(AttributeKey<T> key, T value);// 设置Channel的属性，后文会具体提到

public B handler(ChannelHandler handler)；// 设置添加到pipeline的handler
```

另一类接口是返回`ChannelFuture`的:
```java
public ChannelFuture bind(); // 绑定channel
public ChannelFuture connect(); // 连接到远程节点
```
综上大致有两类接口:
1. 进行全局配置;
2. 进行具体action。

#  Bootstrap: 客户端/无连接服务端
`Bootstrap`一般用于客户端，也可以用于无连接协议的服务端。
程序员可以通过`Bootstrap`上的接口设置一些程序需要的组件具体实现是什么。

两种引导行为:
`bind`: (服务端) 绑定本地服务到某个端口, 然后创建一个Channel，准备接受连接;
`connect`: (客户端) 创建一个Channel，连接远端服务。

简单客户端: 
要点: 
配置: 必须配置好group,channel和handler, 然后`connect`。
// channel或者channelFactory
```java
public void bootstrap() {
        //设置 EventLoopGroup，提供用于处理 Channel 事件的 EventLoop
        EventLoopGroup group = new NioEventLoopGroup();
        //创建一个Bootstrap类的实例以创建和连接新的客户端Channel
        Bootstrap bootstrap = new Bootstrap();
        bootstrap.group(group)
            .channel(NioSocketChannel.class) // NIO
            .handler(new SimpleChannelInboundHandler<ByteBuf>() {
                @Override
                protected void channelRead0(
                    ChannelHandlerContext channelHandlerContext,
                    ByteBuf byteBuf) throws Exception {
                    System.out.println("Received data");
                }
                });
        //连接到远程主机
        ChannelFuture future = bootstrap.connect(
            new InetSocketAddress("xxx.com", 80));
        future.addListener(new ChannelFutureListener() {
            @Override
            public void operationComplete(ChannelFuture channelFuture)
                throws Exception {
                if (channelFuture.isSuccess()) {
                    System.out.println("Connection established");
                } else {
                    System.err.println("Connection attempt failed");
                    channelFuture.cause().printStackTrace();
                }
            }
        });
    }
```

# ServerBootstrap: 服务端
`ServerBootstrap`比普通的`Bootstrap`多了几个接口:
```java
childOption: 后续accept的子Channel的配置; // bind后生效
childAttr: 设置给 已经accept 的子Channel属性; 
childHanlder: 设置给 已经accept 的子Channel的pipeline。
```
作为对比的三个接口:
```java
option: 后续新建的serverChannel的配置; // bind后生效
attr: 设置给当前serverChannel的属性;
handler: 设置给当前serverChannel的pipeline。
```
容易混淆的就是Option用于配置固定的几个参数比如超时时间,Attr用于存自定义属性。

## ServerChannel
accept新连接后,`ServerChannel`创建`子Channel`。
// `子Channel`代表已被接受的连接
{% img /images/2018-09/8.3.png 400 600 serverChannel %} 
工程过程:
1. 调用bind，创建`ServerChannel`, 绑定到本地端口开始提供服务;
2. 接受1个新连接, `ServerChannel`创建1个新的`子Channel`。

简单服务端:
要点: 配置group,channel,childHandler，然后`bind`。
```java
/**
     * 代码清单 8-4 引导服务器
     * */
    public void bootstrap() {
        NioEventLoopGroup group = new NioEventLoopGroup();
        ServerBootstrap bootstrap = new ServerBootstrap();
        // 1. 设置:
        bootstrap.group(group)
            .channel(NioServerSocketChannel.class)
            .childHandler(new SimpleChannelInboundHandler<ByteBuf>() {
                @Override
                protected void channelRead0(ChannelHandlerContext channelHandlerContext,
                    ByteBuf byteBuf) throws Exception {
                    System.out.println("Received data");
                }
            });
        // 2. bind: 通过配置好的 ServerBootstrap 的实例绑定该 Channel
        ChannelFuture future = bootstrap.bind(new InetSocketAddress(8080));
        // 3. listener future:
        future.addListener(new ChannelFutureListener() {
            @Override
            public void operationComplete(ChannelFuture channelFuture)
                throws Exception {
                if (channelFuture.isSuccess()) {
                    System.out.println("Server bound");
                } else {
                    System.err.println("Bind attempt failed");
                    channelFuture.cause().printStackTrace();
                }
            }
        });
    }
```

# 从Channel引导客户端(ServerBootstrap需要connect别的Server时)
假设我们有一个`ServerBootstrap`，提供服务，收到一个请求，发现需要另一个服务的帮助。（相当于我们需要提供代理）
两种解决方案:
1. 再起一个`Bootstrap`，去`connect`另一个服务，获得结果再返回;
2. 和1类似,稍有不同的时, 当前链路复用同一个`EventLoop`。

方案2的优点:
1. 避免创建额外的线程; 
2. 减少上下文切换开销.

示意图如下:
{% img /images/2018-09/8.4.png 400 600 复合bootstrap %}
相关代码如下。
要点其实只有一句话:
`bootstrap.group(ctx.channel().eventLoop());`
这里用`EventLoop`填充`EventLoopGroup`，类似于子类对象填入基类指针。
因为`EventLoop`和`EventLoopGroup`的关系如下:
```java
public interface EventLoop extends OrderedEventExecutor, EventLoopGroup {
    @Override
    EventLoopGroup parent();
}
```
代理服务器代码: 
```java
public void bootstrap() {
        ServerBootstrap bootstrap = new ServerBootstrap();
        // 1. 配置:
        bootstrap.group(new NioEventLoopGroup(), new NioEventLoopGroup())
            .channel(NioServerSocketChannel.class)
            .childHandler(// 注意是childHandler
                new SimpleChannelInboundHandler<ByteBuf>() {
                    ChannelFuture connectFuture; // handler的成员变量
                    @Override
                    public void channelActive(ChannelHandlerContext ctx)
                        throws Exception {
                        // 1.1 新建boostrap
                        Bootstrap bootstrap = new Bootstrap();
                        bootstrap.channel(NioSocketChannel.class).handler(
                        // 注意是Handler
                            new SimpleChannelInboundHandler<ByteBuf>() {
                                @Override
                                protected void channelRead0(
                                    ChannelHandlerContext ctx, ByteBuf in)
                                    throws Exception {
                                    System.out.println("Received data");
                                }
                            });
                        // 1.2 bootstrap配置: 使用channel的eventLoop引用:
                        bootstrap.group(ctx.channel().eventLoop());
                        connectFuture = bootstrap.connect(
                            new InetSocketAddress("xxx.com", 80));
                    }
                    @Override
                    protected void channelRead0(
                        ChannelHandlerContext channelHandlerContext,
                            ByteBuf byteBuf) throws Exception {
                        if (connectFuture.isDone()) { 
                            // 1.3: 当连接完成时，执行一些数据操作（如代理）
                        }
                    }
                });
        
        // 2. bind: 
        ChannelFuture future = bootstrap.bind(new InetSocketAddress(8080));
        future.addListener(new ChannelFutureListener() {
            @Override
            public void operationComplete(ChannelFuture channelFuture)
                throws Exception {
                if (channelFuture.isSuccess()) {
                    System.out.println("Server bound");
                } else {
                    System.err.println("Bind attempt failed");
                    channelFuture.cause().printStackTrace();
                }
            }
        });
    }
```
可以注意到还有一个与之前不同的奇异点:
```java
bootstrap.group(new NioEventLoopGroup(), new NioEventLoopGroup())
```
这里其实为`ServerChannel`和`子Channel`设置了不同的`EventLoopGroup`。以前都是设置成同一个的。
查看源码:
```java
@Override
public ServerBootstrap group(EventLoopGroup group) {
    return group(group, group);
}
 
public ServerBootstrap group(EventLoopGroup parentGroup, EventLoopGroup childGroup) {
    super.group(parentGroup);
    if (childGroup == null) {
        throw new NullPointerException("childGroup");
    }
    if (this.childGroup != null) {
        throw new IllegalStateException("childGroup set already");
    }
    this.childGroup = childGroup;
    return this;
}
```
 
此外，客户端使用`handler`,服务端使用`childhandler`。


# ChannelInitialize
如果有多个`ChannelHandler`，一般会使用`ChannelInitialize`来添加：
1. `childHandler`里传入一个`ChannelInitialize`:
```java
bootstrap.group(new NioEventLoopGroup(), new NioEventLoopGroup())
            .channel(NioServerSocketChannel.class)
            .childHandler(new ChannelInitializerImpl());
```

2. `ChannelInitializerImpl`里依次添加真正的`handler`:
```java
final class ChannelInitializerImpl extends ChannelInitializer<Channel> {
        @Override
        protected void initChannel(Channel ch) throws Exception {
            ChannelPipeline pipeline = ch.pipeline();
            pipeline.addLast(new HttpClientCodec());
            pipeline.addLast(new HttpObjectAggregator(Integer.MAX_VALUE));
        }
    }
```

这里用的是`ChannelInitializer<Channel>`，所以方法会应用到`SocketChannel`和`Channel`。
如果只想应用到`SocketChannel`，可以用`ChannelInitializer<SocketChannel>`。


# ChannelOption,Attr
### ChannelOption:
`bootstrap`的`option`方法可以用来设置之后创建的`channel`公用的配置项:
```java
bootstrap.option(ChannelOption.SO_KEEPALIVE, true)
         .option(ChannelOption.CONNECT_TIMEOUT_MILLIS, 5000);
```
可以设置底层连接的超时时间以及缓冲区设置。

### Attr
`attr`则用来存储一些`Channel`公用自定义属性:
```java
// 1. 声明:
final AttributeKey<Integer> id = AttributeKey.newInstance("ID");
Bootstrap bootstrap = new Bootstrap();
// 2. 赋值:
bootstrap.attr(id, 123456);
// 2. 使用:(某个handler里头:)
@Override
public void channelRegistered(ChannelHandlerContext ctx)
    throws Exception {
    Integer idValue = ctx.channel().attr(id).get();
    // do something with the idValue
}
```

可以看出两者都是整个`bootstrap`共用的，换言之是所有`Channel`共用的。

// 对于服务端来说可以使用`childOption`和`childAttr`。

# DatagramChannel/无连接
FileChannel：文件通道，用于文件的读写
DatagramChannel：用于UDP连接的接收和发送
SocketChannel：TCP客户端
ServerSocketChannel：TCP服务端

```java
// 1. OIO版本:
Bootstrap bootstrap = new Bootstrap();
        bootstrap.group(new OioEventLoopGroup()).channel(
            OioDatagramChannel.class).handler(
            new SimpleChannelInboundHandler<DatagramPacket>() {
                @Override
                public void channelRead0(ChannelHandlerContext ctx,
                    DatagramPacket msg) throws Exception {
                    // Do something with the packet
                }
            }
        );
ChannelFuture future = bootstrap.bind(new InetSocketAddress(0));

// 2. NIO版本
// 服务端:
final NioEventLoopGroup nioEventLoopGroup = new NioEventLoopGroup();
Bootstrap bootstrap = new Bootstrap();
bootstrap.channel(NioDatagramChannel.class);
bootstrap.group(nioEventLoopGroup);
bootstrap.handler(new ChannelInitializer<NioDatagramChannel>() {...});
ChannelFuture sync = bootstrap.bind(9009).sync();
Channel udpChannel = sync.channel();
sync.closeFuture().await();// 等待关闭。
// 客户端:
final NioEventLoopGroup nioEventLoopGroup = new NioEventLoopGroup();
Bootstrap bootstrap = new Bootstrap();
bootstrap.channel(NioDatagramChannel.class);
bootstrap.group(nioEventLoopGroup);
bootstrap.handler(new ChannelInitializer<NioDatagramChannel>() {...});
ChannelFuture sync = bootstrap.bind(0).sync();
Channel udpChannel = sync.channel();
String data = "data";
udpChannel.writeAndFlush(new DatagramPacket(Unpooled.copiedBuffer(data.getBytes(Charset.forName("UTF-8"))), new InetSocketAddress("192.168.2.29", 9008)));
sync.closeFuture().await();// 等待关闭。
```
可以看出客户端和服务端是一样的。
都是用`Bootstrap`,`bind`方法。


# 关闭
```java
 //shutdownGracefully()方法将释放所有的资源，并且关闭所有的当前正在使用中的 Channel
Future<?> future = group.shutdownGracefully();
// block until the group has shutdown
future.syncUninterruptibly();
```

此外之前也可以用`channel.close()`手动关闭一些`channel`。


