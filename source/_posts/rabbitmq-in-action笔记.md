---
title: rabbitmq in action笔记
date: 2017-03-24 19:57:10
tags: rabbitmq
categories:
- 消息队列
- rabbitmq
---

管理面板:(好用)
http://localhost:15672/
基本功能可以在网页上进行,创建vhost等高级功能需要命令行.


`Channel` : 信道.一个连接多个Channel. 节省开销.
`Exchange`: 交换器.  
`Queue`: 队列. 带名字的邮箱. 
`Binding`: 绑定.// 路由规则

基本工作流程:
> 1. 配置类:
创建exchange,queue;
为某个exchange创建binding(路由规则).
比如为exchange1创建规则:
`routing key 为info的,转发到 queue1中.`

2.生产者: 
> 将消息打上routing key,发送到某个exchange中.
如果不指定exchange,默认发送到名为""的exchange中.

3.消费者:
> 从某个queue提取消息.

4.exchange:
> 根据配置好的路由规则,转发收到的消息到符合的queue.




basic.cosume: 订阅
basic.get : 先订阅,然后获取一条,然后取消订阅.

同一个队列多个消费者的话,1个消费者取了消息(ack),消息被删除,别的消费者就取不到了. 因此这种情况应该使用多个队列.(每人有自己的邮箱)

`exclusive`: 声明队列为私有的.(单消费者)
`durable`: 适用于交换器和队列,重启后是否消失.

msg-->`Exchange`->`Queue`

生产者:
msg-->`Exchange`
消费者:
`Queue`->msg
RabbitMQ:
`Exchange`--`binding`-->`Queue`
e.g:
```
binding:
"hello-queue"--"routingkey"-->"hello-exchange"
```


# Exchange
- direct 直连
- fanout 一个消息到多个队列
- topic  多个消息到一个队列 
- header // 不实用,即将弃用

默认`Exchange`名字为空字符串`""`,连接到所有队列.
```
private static final String DEFAULT_EXCHANGE = "";
private static final String DEFAULT_ROUTING_KEY = "";
```


topic的routing key格式:
- `*`: 通配符,`.`以外的任何字符任意个
- `.`: 分隔符
- `#`: 所有队列. 

# vhost (多租户)
默认vhost是`/`. 
每个vhost里可以有自己的`Exchange`,`Queue`,用户,权限等等. 
彼此隔离.

# 持久化消息
1. 投递时指定
2. 投递到持久交换器
3. 投递到持久队列
