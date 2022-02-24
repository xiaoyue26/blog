---
title: http和tcp层面的keepAlive机制
date: 2022-02-24 10:18:55
tag:
- http
- tcp
- keepalive

categories: 
- http

---

# What: keepAlive机制是什么
## HTTP层面
http1.0是短连接，每次http请求都建立tcp连接然后断开；（3次握手4次挥手）
http1.1为了优化性能，推出keepAlive机制，同域名的多个http请求可以复用同一个tcp连接，也就是让tcp连接不每次断开，keepAlive。
对于http协议来说，就是在header里标示这种需求: `Connection: Keep-Alive`

HTTP层面实际做的事情: header里标示keepAlive，然后通信双方不主动关闭tcp连接。
// http没有发送多余的保活、探活报文；

## TCP层面
> 背景
TCP层面本来没必要做keepAlive，毕竟一个连接没有被关闭默认就是alive的。
但因为实际使用的时候，空闲的tcp连接会被 防火墙、负载均衡、代理软件 等中间节点掐断，
而且一般它们掐断空闲tcp连接的时候并不会向客户端发送任何报文提醒，因此客户端是无感知的。
这种情况可能产生 `Broken Pipe`错误。
为了避免这种情况，windows和linux内核在实现tcp的时候，加上了keepAlive机制。

TCP层面的keepAlive实际做的事情: 发送keepAlive报文，返回对端的存活性；
两个作用：
1. 保活: 发送心跳报文，防止tcp连接被识别为空闲连接; // 防止被别人掐断；
2. 探活: 探测对方的存活性，如果对方真的不在了，也不能浪费资源维持连接； // 主动自己断开；

本质上就是：要断开连接的话，还是自己来吧，不让别人代劳了，不然别人掐了也不告诉我，凭空增加了我的异常；



# How: 如何配置tcp层面的keepAlive
## 保活: keepAlive间隔配置
linux内核tcp的keepAlive报文间隔默认是2小时； (`net.ipv4.tcp_keepalive_time`)
实际应该参考常见的"掐断连接凶手"的空闲连接配置，设置短一些。
比如：
F5: 空闲5分钟掐断；
GoogleCloud防火墙：空闲10分钟掐断；

所以可以考虑配置成5分钟keepAlive一次

## 探活: 主动断开相关的配置：
`net.ipv4.tcp_keepalive_intvl`: 对端不正常的话，多久重试1次； 比如75秒；
`net.ipv4.tcpkeepaliveprobes`: 最多重试几次以后断开； 比如9次以后主动断开；

## Http层面KeepAlive配置
HTTP的头部：
1。 End-to-end头部：头部会被中间的代理原样转发；如Host等大部分Header；
2。 Hop-by-hop头部：只到下一跳节点；如KeepAlive头部；
因此要考虑Http请求链路上每一跳(比如Nginx)的KeepAlive配置，才能达到整个http请求涉及到的tcp连接都不自动断开。

# HTTP层面的keepAlive和tcp层面的keepAlive关系
如果没有tcp层面的keepAlive，http层面希望的连接不断开，可能无法实现；
可能被中间的节点（比如防火墙、负载均衡、NAT代理）掐断。

如果tcp层面的keepAlive配置正确，http层面的keepAlive才能正常完成。

# 参考
https://support.f5.com/csp/article/K13004262
https://cloud.tencent.com/developer/news/696654


