---
title: 半小时真正理解记住paxos协议
date: 2019-02-01 09:45:14
tags:
- 分布式
- paxos 
categories: 分布式

---

摘要:
> 简单浏览一遍流程，然后看半小时视频: https://www.bilibili.com/video/av36134550
就OK了。


paxos是分布式一致性协议，对于CAP的取舍是完全的C，较好的A，较好的P。
（关于CAP: http://xiaoyue26.github.io/2019/02/01/2019-02/简单解释CAP原理 ）

# paxos协商流程
## 两个角色
Proposer
Acceptor

## 流程
首先大致扫一遍下叙流程，第一遍大概率是完全懵逼看不懂的，花1分钟在脑中留下一点点印象即可:
### 准备阶段（prepare）Proposer申请epoch
第一阶段A：Proposer选择一个提议编号n，向所有的Acceptor广播Prepare（n）请求。

第一阶段B：若Acceptor接收到Prepare（n）请求，两个选择:
    (1)n>接受过的n_old: 接受请求,返回`(n_old,value_old)`。
    (2)n<接受过的n_old: 拒绝请求。

### 接受阶段（提交阶段）

第二阶段A：Proposer得到了Acceptor响应:
1.如果未超过半数accpetor响应，直接转为提议失败；

2.如果超过多数Acceptor的承诺，又分为不同情况：
(1)如果所有Acceptor返回的value都是`null`，那么向所有的Acceptor发送`(n,value_n)`;
(2)如果存在Acceptor返回的value不为`null`，那么从所有接受过的值中选择对应的n_old最大的作为提议的值value_old，提议编号仍然为n。但此时Proposer就不能提议自己的值，只能信任Acceptor通过的值，维护一但获得确定性取值就不能更改原则: 向所有Acceptor发送`(n,value_old of max n_old)`。

第二阶段B：Acceptor接收到提议`(n,value)`后：
(1)n!=本地保存的last_n，拒绝。
(2)n=本地保存的last_n，写入本地值。

## 看教学视频:
https://www.bilibili.com/video/av36134550

由于之前花1分钟浏览了流程，视频看到15分钟的时候心里就会已经有数了。
看完30分钟就理解记住了。

## 温习要点
1. 类似于CAS，仅当，当前值为null时，才能提议值为value。
2. 类似于jdk里的hashmap扩容，当发现有人捷足先登的时候，放下陈见开始辅助n_old完成value_old的扩散。发送(n,value_old)，以自己的名义，但是以别人的值，帮助一致性尽快达成。
3. Acceptor对于epoch: 喜新厌旧,总是接受最大的epoch，防止Proposor单点故障死等。
4. Proposor对于value: 喜旧厌新,总是接受更旧的value，尽快达成一致性。

以上就是Basic Paxos。其实依然会有活锁，需要引进leader缓解。leader挂了的时候退化到Basic Paxos，出现活锁几率提升。










