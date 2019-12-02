---
title: RTP/RTCP协议
date: 2019-12-02 10:09:59
tags: RTP
categories: 
- 网络

---


# 实时传输协议RTP/RTCP
参考:
https://www.jianshu.com/p/631273bc9847

底层理论上也不一定用tcp/udp,实际上为了性能大多用udp。

RTP:  `Real-time Transport Protocol`
RTCP: `RTP Control Protocol`
{% img /images/2019-12/RTCP.jpg 800 1200 RTCP %}

```
RTP: 用udp来传数据；      (偶数端口)
RTCP: 用udp来传控制信息； (奇数端口, 一般是RTP的端口+1) 服务质量的监视与反馈、媒体间的同步，以及多播组中成员的标识
```

## 计算机网络
交换机级别： 组播协议，一对多
多对多可以转换成一对多。