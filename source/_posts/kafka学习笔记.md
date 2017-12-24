---
title: kafka学习笔记
date: 2017-03-01 19:53:02
tags: kafka
categories: 
- 消息队列
- kafka
---

原理:(与rocketMQ对比,理解更深刻)
http://blog.csdn.net/chunlongyu/article/details/54018010

quickStart:
http://kafka.apache.org/quickstart

照着官网做就好了.起步简单.

#Master/Slave概念差异
Kafka:
> Master/Slave是个逻辑概念，1台机器，同时具有Master角色和Slave角色。 

RocketMQ:
> Master/Slave是个物理概念，1台机器，只能是Master或者Slave。在集群初始配置的时候，指定死的。其中Master的broker id = 0，Slave的broker id > 0。

#Broker概念差异
Kafka:
 > Broker是个物理概念，1个broker就对应1台机器。 

RocketMQ：
> Broker是个逻辑概念，1个broker = 1个master + 多个slave。所以才有master broker, slave broker这样的概念。
那这里，master和slave是如何配对的呢？ 答案是通过broker name。具有同1个broker name的master和slave进行配对。

RocketMQ不依赖ZK,是因为设计上进行了简化,不需要那么多选举.
用个简单的NameServer就搞定了，很轻量，还无状态，可靠性也能得到很好保证。