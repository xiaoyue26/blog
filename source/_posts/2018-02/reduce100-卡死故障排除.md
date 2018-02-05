---
title: reduce100%卡死故障排除
date: 2018-02-05 10:28:17
tags: hadoop
categories:
- hadoop
- 运维

---

- 摘要
> 集群所有MR job/Spark job奇慢无比,经过排查发现两个DataNode故障.
措施:
下线故障的DataNode,所有job运行重新变得飞快顺滑.
原因:
reduce结束后落盘的时候,pipeline中其中一个DN故障,导致返回太迟.
如果pipeline中恰好不包括故障DN,就能顺利完成.
(因此延迟时长比例与故障数正相关.)
排查三板斧:
1. 杀job;
2. 杀job;
3. 杀DN.

## 详情
早上起来发现集群的任务只完成了平时的约1/8.
### STEP1
首先怀疑是调度不合理,job之间资源占用卡死.
进入管理页:http://f04:8088/cluster/scheduler
找到卡死的job,选出资源占用最大的job将其kill. 

表现1:
资源空闲情况下,即使是小job,也会在reduce的最终阶段卡死不动.最显著的特征是在reduce100%的状态下,依然卡死不动.
查看典型job后发现有如下log:
```
WARN [ResponseProcessor for block BP-133847077-ip0-1413174984773:blk_3659724856_2615805444]
org.apache.hadoop.hdfs.DFSClient: Slow ReadProcessor read fields took 54565ms (threshold=30000ms);
ack: seqno: 144299 reply: SUCCESS reply: SUCCESS reply: SUCCESS downstreamAckTimeNanos: 61280175650
flag: 0 flag: 0 flag: 0, targets:
[DatanodeInfoWithStorage[ip1:50010,DS-7048c932-5153-4b82-8f6e-e8a920f61947,DISK]
, DatanodeInfoWithStorage[ip2:50010,DS-f11ae138-54eb-4fda-aeda-43e3a9eddeae,DISK]
, DatanodeInfoWithStorage[ip3:50010,DS-82512645-0f03-434f-8693-4f1c8876c437,DISK]]
```
虽然只是一个`Warning`,但其实关键信息是reduce最后落盘的时候落不下来,写数据块的时候`pipeline`里的三个DN,至少有一个迟迟不接Block.

进一步登录上述三个DN(ip1,ip2,ip3),查看DN的log,发现大致有如下log:
```
INFO datanode.DataNode: Exception for BP-133847077-ip0-1413174984773:blk_3659234296_2615314111
java.io.InterruptedIOException: Interrupted while waiting for IO on channel java.nio.channels.SocketChannel[connected
local=/ip4:24281 remote=/ip1:50010]. 485000 millis timeout left.
	at org.apache.hadoop.net.SocketIOWithTimeout$SelectorPool.select(SocketIOWithTimeout.java:342)
	at org.apache.hadoop.net.SocketIOWithTimeout.doIO(SocketIOWithTimeout.java:157)
	at org.apache.hadoop.net.SocketOutputStream.write(SocketOutputStream.java:159)
	at org.apache.hadoop.net.SocketOutputStream.write(SocketOutputStream.java:117)
	at java.io.BufferedOutputStream.write(BufferedOutputStream.java:122)
	at java.io.DataOutputStream.write(DataOutputStream.java:107)
	at org.apache.hadoop.hdfs.protocol.datatransfer.PacketReceiver.mirrorPacketTo(PacketReceiver.java:200)
	at org.apache.hadoop.hdfs.server.datanode.BlockReceiver.receivePacket(BlockReceiver.java:556)
	at org.apache.hadoop.hdfs.server.datanode.BlockReceiver.receiveBlock(BlockReceiver.java:895)
	at org.apache.hadoop.hdfs.server.datanode.DataXceiver.writeBlock(DataXceiver.java:801)
	at org.apache.hadoop.hdfs.protocol.datatransfer.Receiver.opWriteBlock(Receiver.java:137)
	at org.apache.hadoop.hdfs.protocol.datatransfer.Receiver.processOp(Receiver.java:74)
	at org.apache.hadoop.hdfs.server.datanode.DataXceiver.run(DataXceiver.java:253)
	at java.lang.Thread.run(Thread.java:745)
```

查看DN的连接数,发现异乎寻常得高:
```
netstat -n | awk '/^tcp/ {++S[$NF]} END {for(a in S) print a, S[a]}'
```

## STEP2
经过收集几个卡住的job的log,统计pipeline中出现的DN的ip,可以找出出现频率最高的两个DN.
将上述DN kill,卡住的job就能够继续运行了.
> pipeline中某个DN挂了的话,会提前返回. 三备份会在以后找机会补上.

下线(退役)DN,其上的Block会自动备份到其他DN.
集群故障排除.具体DN故障的原因待查.// TODO


