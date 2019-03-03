---
title: redis设计与实现笔记9-cluster模式
date: 2019-03-03 21:05:48
tags:
- redis
categories:
- redis

---

cluster功能是3.0及以后才有的。需要开启cluster模式才能让redis-server以cluster模式启动，这种模式下只有一个数据库（0号数据库）。

启动集群模式的redis客户端:
```sh
# 加上-c参数: 
redis-cli -c
```

## 集群纳入新成员过程
1. 节点A接到Cluster meet B命令，节点A和B进行握手；
2. 节点A会将节点B的信息通过`Gossip`协议传播给集群中的其他节点；
3. 最终节点B被集群完全认识、接受。

## Gossip协议消息类型
1. Meet;
2. Ping;
3. Pong。


## slot: 槽指派
某个节点负责哪些slot:
```sh
127.0.0.1> cluster ADDslots <slot> [slot ...]
```

## Cluster Nodes命令
用redis-cli客户端可以查看当前集群的节点情况、id，以及slot的分派情况：
```shell
Cluster Nodes
```

## clusterState信息
clusterState信息中有一项是`slots_to_keys`跳表(类似一个有序hashmap)，保存slot和key之间的关联。
key=>跳表中的key;
slot号码=>跳表中的score。

key查节点、节点查key都可以快速完成:
1. 每个节点可以根据key，查到对应的slot(crc算法)，然后可以查到对应存在哪个节点（存在跳表）；
2. 可以查自己节点负责的slot:
```c
struct clusterNode{
    // 总共16384(也就是2^14)位，每一位的1,0代表是否负责
    unsigned char slots[16384/8];
    int numslots;// 该节点总共负责多少个slot(1的数量)
}
```

`clusterState`中存储了所有slots的指派情况:
```c
typedef struct clusterState{
    // ...
    clusterNode* slots[16384];
}clusterState;
```

如果集群中所有的slot都有人负责，cluster进入上线状态。

## Moved错误
客户端访问了错误的节点，需要的key在别的节点负责的slot里头：
```shell
MOVED <slot> <ip>:<port>
```
cluster模式的redis客户端能自动处理`MOVED`错误。

## ASK错误
### 重新指派
{% img /images/2019-03/cluster_migrate.png 800 1200 cluster_migrate %}
集群可以将slot重新分派给另一个节点。
也就是: 源节点=迁移到=>目标节点。
其中节点id可以通过`cluster nodes`获得。
通过redis-trib工具：
1.在target_id节点上发送: (准备好导入)
```sh
Cluster setslot <slot> Importing <source_id>
```
2.向source_id发送:（准备好导出）
```sh
Cluster SetSlot <slot> Migrating <target_id>
```
3.向source_id发送:
```sh
Cluster GETkeysInSlot <slot> <count>
```
获得最多count个属于slot的key；
4.对于第3步中的每个key，redis-trib向source_id发送一个命令:
```sh
Migrate <target_ip> <target_port> <key_name> 0 <timeout>
```
实际进行迁移每一个key。
5.完成迁移，发布:
```sh
Cluster SetSlot <slot> Node <target_id>
```
任意节点收到后传播给整个集群。

### ASK错误
{% img /images/2019-03/ask_error.png 800 1200 ask_error %}
迁移过程中，源节点遇到缺少的key会向客户端返回ASK错误:
```
ASK 16198 127.0.0.1:7003
```
表示这个key正在将16198号slot迁移到`127.0.0.1:7003`。

{% img /images/2019-03/ask_query.png 800 1200 ask_query %}
cluster模式的客户端获得ASK错误后，带着ASK标记去访问目标节点，才能获得数据。

### 故障恢复
节点状态：
`在线`==>`疑似下线`==>`下线`
(类似于之前`sentinel`模式中的`主观下线`和`客观下线`)。

cluster模式的各个主节点之间都有连接，相当于一张完全图了。
这个网络内，它们每隔一段时间就互相ping，看看对方是否活着（返回pong）。

`疑似下线`： 不返回pong，标记为`疑似下线`；
`下线`    ： 节点之间交流看法(集群状态)，超过半数认为`疑似下线`则认为确实是`下线`，开始故障恢复。


如果离线的节点有从节点，则可以开始选举了。
从节点们发现主节点下线了，就开始向集群存活的主节点们请求投票，获得超过半数的票则当选。

## Publish命令实现
订阅频道：      subscribe
发布消息到频道：publish

客户端发布消息的流程：
1. 客户端=>某个节点: publish命令: 发布xx消息到xxx频道;
2. 该节点=>其他节点: publish的具体消息。

第2步中为什么不是直接转发命令到其他节点，而是转发消息呢？
这个主要是设计理念上，希望节点之间不是命令交互，而是消息交互。