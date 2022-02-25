---
title: quic中的拥塞算法bbr
date: 2022-02-18 19:01:21
tag:
- http
- java
- spring
- bbr
- cubic
- quic
categories: 
- http

---

# BBR
BBR = `Bottleneck Bandwidth and Round-trip time` 

# 拥塞控制算法
tcp默认使用cubic算法做拥塞控制;
谷歌的quic开发了新的拥塞控制算法，吞吐量更大，更能利用带宽开发了新的拥塞控制算法`bbr`,`bbr_v2`;

# CUBIC vs BBR
cubic vs bbr:
{% img /images/2022-02/cubic_vs_bbr.png 400 600 cubic_vs_bbr %}
可以看到tcp默认的cubic拥塞控制算法频繁上下调整滑动窗口大小，锯齿状；
而bbr倾向于平稳发送，在实际带宽比较平稳的场景下，吞吐量更大。（图中折线下方的面积更大）

原来tcp为什么没有解决这个问题：
(1)tcp在linux内核里，升级太困难了。
(2)tcp的一些约束导致rtt算不准。比如ack delay、重传包的seq number不变。

> cubic: 基于丢包；锯齿形吞吐；事件驱动； tcp的重传包seqId不变，rtt算不准；
> BBR: 基于延迟； 有平滑区间；根据rtt建立对带宽（窗口大小）的模型，再加上定时器；
> quic的重传包，seqId增加，rtt算得准。区分了具体的重传类型。
> 需要注意：tcp的ack delay时间影响rtt计算；(默认40ms)

BBR的思想：
当rtt开始增长的时候，就达到了最大带宽。

cubic：把缓存塞满一直到丢包；
对丢包率的容忍非常低，即使只有极少的丢包，吞吐量也会急剧下降。

> 缓冲膨胀
> 指的网络设备或者系统不必要地设计了过大的缓冲区。
> 当网络链路拥塞时，就会发生缓冲膨胀，从而导致数据包在这些超大缓冲区中长时间排队。
> 在先进先出队列系统中，过大的缓冲区会导致更长的队列和更高的延迟，并且不会提高网络吞吐量。
> 由于BBR并不会试图填满缓冲区，所以在避免缓冲区膨胀方面往往会有更好的表现。


弱网环境,引入1.5%的丢包：
cubic：吞吐量下降99.7%
bbr: 吞吐量下降45%

# BBR的缺点
1。wifi环境网速变慢；
2。网络公平性下降: 挤占cubic算法带宽；
3。重传率会更高；
4。bbr对于rrt的测量是包级别的，可能容易受波动影响，可以考虑统计学上进行优化；

bbr的平稳发送时期本质上假设网络环境有一段时间是平稳的，因此比cubic抖动少，大部分情况下实际情况确实如此。

# 参考资料
https://aws.amazon.com/cn/blogs/china/talking-about-network-optimization-from-the-flow-control-algorithm/
bbr思想：https://blog.csdn.net/dog250/article/details/52962727






