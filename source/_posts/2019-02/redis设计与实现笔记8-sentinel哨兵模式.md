---
title: redis设计与实现笔记8-sentinel哨兵模式
date: 2019-02-24 20:46:55
tags:
- redis
categories:
- redis

---

## 使用场景:
用几个节点开启sentinel组成一个哨兵集群，负责监控另外一些redis的master/slave集群的健康状态，协助进行故障恢复(master挂了的时候，升级某个slave为新master)。

## 思考:
如果有1个master,10个slave，数据均已经完全同步。
这个时候，连续挂两次master，是不是所有数据就都没了？
第1次: 除新master以外的节点，执行新slaveof命令，清空数据准备同步新master;
第2次: 新master挂了，其他节点是空的。

## 答案：
不会这么脆弱。
两个机制提升了这个过程的可靠性：
(1)确认接受到完整rdb后，从库才清空旧数据库;
(2)确认所有从库完成同步后，才更新master的地址和端口，完成故障恢复流程。（master/slave换代操作会在故障恢复完全完成后进行。）

第1次： 除候选master以外的节点，执行slaveof命令，但清空数据会在确认接收到完整rdb文件后进行。（详见代码https://github.com/huangz1990/redis-3.0-annotated/blob/unstable/src/replication.c的1036行。）
第2次：候选master挂了，重新推举候选master，换代没有完全完成，则不会更新master字段，因此其他slave都还在候选集中。


相关参数: 
```sh
# 执行故障转移操作时，可以同时对新master进行同步的从库数量:
SENTINEL parallel-syncs <master-name> <number>
```

# 哨兵模式:Sentinel
{% img /images/2019-02/sentinel.png 800 1200 sentinel %}
redis的高可用模式，一主多从+sentinel集群进行监控和故障恢复，主挂的时间达到设置，则选取一个从库升级为主库。

## sentinel启动命令
一个节点用redis代码可以用3种身份(模式)启动：
1. master: 负责写命令；
2. slave: 负责同步、从库，可以执行读命令；
3. sentinel：负责监控上述两者，进行故障恢复。
使用哨兵模式：
```sh
redis-sentinel sentinel.conf
# 或:
redis-server sentinel.conf --sentinel
```
之后发生的事情:
1. 初始化服务器：不载入rdb,aof;（因为不需要负责实际数据）
2. redis服务器切换成Sentinel专用代码;（默认端口26379，只载入部分命令表,客户端只能执行7个命令: 
```
PING
SENTINEL
INFO
Subscribe
unSubscribe
PSubscribe
PUnSubscribe
```
3.初始化sentinel状态；
4.根据配置文件，初始化Sentinel的主库列表；
5.创建与主库的网络连接。


## sentinel相关的网络连接图
引入sentinel后的redis主从架构网络连接较多：
1. sentinel节点与master: 命令连接+订阅连接;
2. sentinel节点与slave:  命令连接+订阅连接;
3. sentinel节点与sentinel: 命令连接。
相关的连接图如下:
{% img /images/2019-02/sentinel-master.png 800 1200 sentinel-master %}
**master地址与端口**：`需要配置`
sentinel需要订阅master的心跳，同时在需要的时候向master发送命令，因此需要两种连接：订阅连接+命令连接。
 
{% img /images/2019-02/sentinel-slave.png 800 1200 sentinel-slave %}
**slave地址与端口**：`不需要配置`
sentinel通过master获取到slave的地址与端口，因此不需要给sentinel配置slavel信息了。
sentinel需要订阅slave的心跳，同时在需要的时候向slave发送命令，因此需要两种连接：订阅连接+命令连接。（同master类似）
{% img /images/2019-02/sentinel-sentinel.png 800 1200 sentinel-sentinel %}
**其他sentinel的地址与端口**: `不需要配置`
sentinel通过master获取到其他sentinel的地址与端口，因此不需要给sentinel配置信息了。sentinel订阅频道的信息里有连接到同一个master的sentinel信息。

由于心跳消息由master帮sentinel完成了，不需要再订阅其他sentinel的心跳了。
每两个sentinel之间都有双向的命令连接（完全图），方便互相发送命令。（客观下线、主观下线、选举leader等命令）


## 一份可能的sentinel配置文件
```conf
## master1 conf:
sentinel monitor master1 127.0.0.1 6379 2 # 需要2票(quorum)才能客观下线
sentinel down-after-milliseconds master1 30000 # 30秒才算主观下线(包括master/slave和其他sentinel)
sentinel parallel-syncs master1 1  # 同时可以有1个从库进行同步
sentinel failover-timeout master1 90000 # 刷新故障迁移状态的最大时限
## master2 conf:
sentinel monitor master2 127.0.0.1 12345 5 # 需要5票才能客观下线
sentinel down-after-milliseconds master2 50000
sentinel parallel-syncs master2 5 
sentinel failover-timeout master2 450000
```

### 初始化sentinel状态
```c
struct sentinelState{
    // 当前纪元
    uint64_t current_epoch;
    // 保存所有被这个sentinel监视的master:
    dict *master;// <master_name,sentinelRedisInstance>
}sentinel;
```
其中master值`sentinelRedisInstance`的结构如下:
```c
typedef struct sentinelRedisInstance{
    // 实例类型、状态:
    int flags;
    char * name; // "127.0.0.1:26379"
    char * runid; 
    uint64_t config_epoch;
    // 实例的地址:
    sentinelAddr *addr;// ip,port
    // 无响应多少毫秒后判断为主观下线:
    mstime_t down_after_period;
    // 判断客观下线所需的支持票数:
    int quorum;
    // 故障转移时,可以同时对新主服务器进行同步的从服务器数量:
    int parallel_syncs;
    // 刷新故障迁移状态的最大时限:
    mstime_t failover_timeout;
}sentinelRedisInstance;
```

### 创建网络连接
sentinel向**每个**监视的master创建两个连接:
1. 命令连接: 用于向master发送、接受命令;
2. 订阅连接: 订阅master的__sentinel__:hello消息。

## 命令连接与订阅连接
### 连向master/slave的命令连接： 4种命令：
1. 每10秒一次的INFO命令：获取master和slave的最新配置信息；
2. 每2秒一次的订阅命令: 获取__sentinel__:hello频道信息，得到其他sentinel的信息。
3. 每1秒一次的PING命令:
获取master/slave/sentinel的心跳信息。
4. 故障恢复的时候的slaveof命令。

### 订阅连接:
master/slave向所有sentinel发送它们订阅的__sentinel__:hello频道信息。

### sentinel之间的命令连接
用于检查客观下线、选举leader、故障恢复。
故障恢复流程：
1. `主观下线`: 某个sentinel用ping命令检查master: 超过`down-after-milliseconds`配置没有回复，该sentinel主观地认为：这个master挂了——它把这个master标记为主观下线状态；
2. `客观下线`: 这个sentinel通过`is-master-down-by-addr`命令询问其他sentinel的意见。超过`quorum`数量sentinel同意，则进入客观下线状态；
3. `选举leader`: leader负责接下来的故障恢复。每次选举结束后(无论成败)，`epoch`纪元都会+1。进入客观下线分支的sentinel会要求其他人选自己，同时它会投第一个向自己要求选票的sentinel一票。所有sentinel会回复其他sentinel自己的选择，因此大家都能确定有谁的票数过半，或者都没有过半，也就是leader选举的成败是确定可知的。（奇数个sentinel的raft算法）
4. `leader`选取新候选master: 
（1）下线原master；（但master结构中依然记录旧地址、端口，不急着更新）
（2）断开候选者slaveof;
（3）其他slave执行slaveof候选者；（同步并行度由参数决定）
（4）当其他slave完成同步，正式任命候选者为master，更新信息到内存。见代码https://github.com/huangz1990/redis-3.0-annotated/blob/unstable/src/sentinel.c中`sentinelHandleDictOfRedisInstances`函数和`sentinelFailoverSwitchToPromotedSlave`函数。
这个过程中如果候选者挂了，会重新选一个候选者。

```c
void sentinelHandleDictOfRedisInstances(dict *instances) {
    dictIterator *di;
    dictEntry *de;
    sentinelRedisInstance *switch_to_promoted = NULL;

    /* There are a number of things we need to perform against every master. */
    // 遍历多个实例，这些实例可以是多个主服务器、多个从服务器或者多个 sentinel
    di = dictGetIterator(instances);
    while((de = dictNext(di)) != NULL) {

        // 取出实例对应的实例结构
        sentinelRedisInstance *ri = dictGetVal(de);

        // 执行调度操作
        sentinelHandleRedisInstance(ri);

        // 如果被遍历的是主服务器，那么递归地遍历该主服务器的所有从服务器
        // 以及所有 sentinel
        if (ri->flags & SRI_MASTER) {

            // 所有从服务器
            sentinelHandleDictOfRedisInstances(ri->slaves);

            // 所有 sentinel
            sentinelHandleDictOfRedisInstances(ri->sentinels);

            // 对已下线主服务器（ri）的故障迁移已经完成
            // ri 的所有从服务器都已经同步到新主服务器
            if (ri->failover_state == SENTINEL_FAILOVER_STATE_UPDATE_CONFIG) {
                // 已选出新的主服务器
                switch_to_promoted = ri;
            }
        }
    }

    // 将原主服务器（已下线）从主服务器表格中移除，并使用新主服务器代替它
    if (switch_to_promoted)
        sentinelFailoverSwitchToPromotedSlave(switch_to_promoted);

    dictReleaseIterator(di);
}
```


选取候选者的大致逻辑：
1. 删除网络条件差的；
2. 考虑因素的顺序：优先级、复制偏移量、运行ID小的。
详见https://github.com/huangz1990/redis-3.0-annotated/blob/unstable/src/sentinel.c代码中的`sentinelSelectSlave`函数。