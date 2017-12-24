---
title: HDFS namenode
date: 2017-01-07 19:25:41
tags: 
- hdfs
- hadoop
categories:
- hadoop
---

# HDFS namenode

1. 双缓冲机制
2. paxos协议

- `hdfs-site.xml`配置文件
```xml
 <property>
    <name>dfs.nameservices</name>
    <value>sync</value>
    <description>Logical name for this new nameservice</description>
  </property>
  <property>
    <name>dfs.namenode.name.dir</name>
    <value>file://home/wudi/hadoop/nn</value>
  </property>
  <property>
    <name>dfs.namenode.shared.edits.dir</name>
    <value>qjournal://host1:port1;host2:port2;host3:port3/sync</value>
  </property>
```
这个配置是说，edit log需要往两个地方写：
1. file://home/wudi/hadoop/nn
2. qjournal://host1:port1;host2:port2;host3:port3/sync

qjournal是一个共享的edit log目录；
JouralNode节点之间运行Paxos协议(帕克萨斯),保证多点写时也能对edit log达成一致。

- 双buffer机制
edit log 维护双buffer,用途：
1. 填充数据；
2. flush。
往buffer写需要事先加锁，写完后检查如果buffer中数据大小达到阈值，则进行sync，将buffer真正写出；
或：线程主动调用sync,主动将buffer写出。
