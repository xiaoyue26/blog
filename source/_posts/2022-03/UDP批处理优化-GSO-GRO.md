---
title: UDP批处理优化-GSO/GRO
date: 2022-03-13 09:59:14
tag:
- udp
- gso
- gpo
categories: 
- http

---

# GSO和GRO
GSO: `Generic Segmentation Offload`
发送端；推迟发送端网络包分片，提高大包发送性能；
GRO: `Generic Receive Offload`
接受端；提早合并到达网卡的小包；提高接受性能；

## GSO原理
before:
刚开始就分片，每个小分片参与后续的IP层、链路层、网卡驱动的系统调用；
after:
在网卡发送前再分片，减少系统调用。
// 1。 网卡支持：由网卡分片；
// 2。 网卡不支持：发送到网卡前一刻，内核做分片；

# GRO
GRO类似于GSO，都是在离网卡最近的地方做分片、合并之类的事情。（批处理）

# 效果
cpu消耗：降低
网络吞吐优化：25%
系统调用次数降低：97%
系统耗时降低：90%
