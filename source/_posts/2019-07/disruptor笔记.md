---
title: disruptor笔记——代替blockingQueue和java9flowAPI
date: 2019-07-27 15:49:48
tags: 
- disruptor
- java
- 消息队列

categories: 
- 消息队列
- disruptor

---

# 背景
blockingQueue缺点:
1. 竞态严重: producer\consumer;
2. cache不友好: 线程a结束后，线程b的所有缓存都污染。

# discruptor方案
理想适用场景：
1个producer,多个consumer.

循环队列：
producer: 存储自己的游标(cursor);
consumer: 存储自己的消费offset。 
1. 循环队列: 降低竞态,分离了多个游标;(当然还是有竞态，用内存屏障)
2. cursor中加入padding: 避免cursor缓存污染;（填充上7个long）
3. 用CAS和忙等(busy spin)代替锁。(资源换性能)
4. 预先申请了一大片内存：避免gc干扰。

- 优化
1. batch commit; // 避免竞态次数 比如消费者不是每次只读1个，它直接询问生产进度seqK，保存下来，然后一直消费到seqK之前都不用再询问cursor. （询问需要穿过consumer barrier）
2. 多个producer时: CAS写入; 
3. 支持复杂dag优化。

附:
```java
// padding:
public final static class VolatileLong
    {
        public volatile long value = 0L;
        public long p1, p2, p3, p4, p5, p6; // comment out
    }
```

## 信息交流
### consumer获取producer信息
{% img /images/2019-07/consumer_barrier.png 400 600 consumer_barrier %}
通过consumer barrier从获取cursor信息： 最新生产sequence number;
这种策略下，消费者不需要知道其他消费者的情况(独立offset)
## producer获取信息
{% img /images/2019-07/producer_barrier.png 400 600 producer_barrier %}
通过producer barrier获取ring buffer和消费者信息。
等待最慢的消费者释放空间：获取最慢的消费者的offset。以便获得更多可读的节点空间。（这里也可以批处理，同时获得多个空间，同时写）

两阶段提交：
1. 数据写入节点;
2. commit. 

# 复杂dag支持
{% img /images/2019-07/disruptor_dag.png 400 600 disruptor_dag %}
如上图所示的菱形结构可能在实际业务中会出现。
producer进度: 22
c1进度: 21
c2进度: 18
c3进度: 15
此时producer想要继续写的时候卡住，因为15的位置还不可用。
也就是生产者的速度受制于叶子节点的消费者。

c1,c2的处理结果一般是原地写入entry的不同字段，避免冲突。
(entry节点的定义可以有多个值字段)

更多详情直接参见：
http://wiki.jikexueyuan.com/project/disruptor-getting-started/write-ringbuffer.html

maven依赖：
https://mvnrepository.com/artifact/com.lmax/disruptor/3.4.2
