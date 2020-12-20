---
title: 升级HTTP2笔记
date: 2020-12-19 15:45:17
tags:
- http
- nginx
categories: http

---


# 已知的坑
## header大小写
header names按http1.1协议是不区分大小写的，http2里全是小写，nginx反向代理会保留大小写，所以如果以前的代码依赖大写，就会挂掉。

(为啥h2里变小写了: HPACK算法, 解压的时候查表还原header names, 类似于哈夫曼算法)

## 与websocket不能共用域名, 只支持https
websocket和http2都是从http协议握手协商升级过去的，也就是起点是http1.1。 两种协商过程:
- http1.1 -Upgrade=> websocket+ssl (wss)
- http1.1 -Upgrade=> http2

H2是基于https的，因此如果某个域名切换到H2以后，就只能支持https的链接了，不再支持http;
如果硬要不加密、不安全，可以使用H2C，但是主流浏览器都声称不会支持H2C，因此这个选项其实并不实际。

 (这个倒是不能算坑，只能算一个特性)
 
# Why: 为什么要升级到H2
主要是性能上的优化。

Http2的修改:

1.header压缩: HPACK算法;

2.服务器推送: server push，推送html里的css,js; 

3.pipeline请求;

4.多路复用，同域名单个TCP连接，划分stream id;

5.二进制传输;

其中1、3、5肯定提升性能；

2则取决于缓存策略，因为可能服务器push了客户端已经缓存的资源，浪费带宽；

4取决于优先级策略，因为把以前前端手动控制的优先级策略，交给浏览器内核来自动实现，按https://blog.cloudflare.com/zh/better-http-2-prioritization-for-a-faster-web-zh/ ， 谷歌内核是最接近最优策略的。但是由于以前可以多开TCP连接，现在是单开，因此如果有大图片，会阻塞后面的小图片。

谷歌的优先级处理逻辑如下：

{% img /images/2020-12/chrome_loading-1.gif 800 1200 loading %}

此外，由于有了特性4，不再需要前端内联资源，因此一些针对http1.1的优化可以回滚，好处是可以简化代码，提高缓存效率，去掉重复建立连接的开销；

坏处就是不能再依赖多开TCP连接了，图片只能一张一张刷开。

# HOW:怎么升级到H2

根据nginx官网的指导，没有必要全链路H2,只需要client到nginx是H2就够了(terminate protocol)。后端服务可以维持原来的协议。（类似于以前升级https）

因此只需要修改nginx配置即可，对后端服务无感知。(如下图)

{% img /images/2020-12/ng.png 800 1200 ng %}

ng官网指导: https://www.nginx.com/blog/7-tips-for-faster-http2-performance/

## 为什么没必要全链路H2:
按ng官网的说法，H2的主要优点是性能提高，对于内网网速来说，这点提升意义不大；

因此nginx只在服务端支持H2, 不支持客户端H2(转发的时候)。

如果想要全链路H2, 就不能用nginx了。


# 参考资料：
1.https://zhuanlan.zhihu.com/p/276057825

2.https://www.cnblogs.com/confach/p/10141273.html

3.https://zhuanlan.zhihu.com/p/89471776

4.https://juejin.im/post/6844903745218674695

5.https://blog.cloudflare.com/zh/better-http-2-prioritization-for-a-faster-web-zh/

6.https://hpbn.co/http2/#stream-prioritization

7.https://calendar.perfplanet.com/2018/http2-prioritization/

8.https://www.jianshu.com/p/e57ca4fec26f

9.https://zhuanlan.zhihu.com/p/26559480

10.https://www.cnblogs.com/ranFengHua/p/10816956.html

11.https://blog.csdn.net/liujiyong7/article/details/64478317

12.https://www.nginx.com/blog/7-tips-for-faster-http2-performance/

13.https://www.cnblogs.com/operationhome/p/12577540.html





 
 



