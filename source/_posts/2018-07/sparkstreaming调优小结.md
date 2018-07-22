---
title: sparkstreaming调优小结
date: 2018-07-22 21:10:41
tags: 
- spark
- streaming
categories:
- spark
- streaming

---



需求：
> 1. 消费速度必须高于生产速度;(天哪)
2. 即使有峰值也不能挂;
3. 资源占用尽量少，其中cores比内存稀缺;
4. 必须使用receiver模式，不提供direct接口。(天哪)

措施:
> 1. receiver数量与分区数一致，与executor数一致； 
2. duartion调优；
3. persist选memory_and_disk；
4. 每个executor分3个core、6g内存;
5. 代码对于每个partition保存一次数据;
6. 关闭backpressure.enabled;
7. receiver.maxRate设得超大。

可能挂的场景：
> 1. 处理速度跟不上，缓存的数据失效了，rdd失联退出；
2. delay越积越多，OOM。



1. `maxRate`与`backpressure.enabled false`
由于底层架构不可控，api被别的部门重新封装，只开放了receiver模式，而且也不给zk连，因此只能使用receiver模式。

这个模式的特点就是会先保存一份数据到wal，相当于所有数据会有两份。
当然也有优点就是编程简单，不用自己维护offset了。

为了保证消费速度高于生产速度，设置了与分区数一致的receiver数量，并且：
```
backpressure.enabled false
receiver.maxRate	149600
```
前者true的话，意思是发现处理不过来的时候，会帮忙降低消费的速度；
后者就是设置最大消费速度。

2. 为了使处理速度够快，设置了3个核，6g内存，相当于每个核2g内存。
由于receiver占用一个核，相当于每个executor上能有两个核同时处理task。

3. 由于receiver模式接受到数据会首先缓存一份rdd。
默认所有rdd会继承dstream的缓存级别，而dstream的缓存级别默认是`MEMORY_ONLY_SER`。（题外话，spark任务的rdd默认缓存级别是`memory_only`）
`MEMORY_ONLY_SER`级别的特点是占用内存少，而牺牲一点计算时间。
由于我们比较不缺内存，因此将存储级别改成`memory_and_disk`。
两个改变：
（1） 取消了反序列化的时间；
（2） 存不下的放disk，应对峰值可能出现的极端情况。

4. duration调优。
duration设置得小的话，内存占用小（缓存rdd），但是提交任务频繁，默认的启动计算开销占比大。
duration设置得大的话，内存占用大（缓存rdd），启动计算开销占比小。

最小资源占比：处理时间正好稍小于duration。
我的设置： 平时处理时间是duartion的1/3，以应对峰值。

5. 保存数据时候的代码优化。
（1）每个partition保存一次；
（2）在executor上保存：
试了一下collect到driver上保存，发现慢了很多，出现了单点瓶颈。于是还是改到在每个executor上保存，每个executor上的线程访问自己的`threadlocal`对象，减少竞态条件。

6. gc时间：
使用CMS收集器，查看exector的gc日志，2小时内没有full gc,3秒一次gc。
gc总时间占task时间的4%：
```
XX:MaxPermSize=256m -XX:SurvivorRatio=4 -XX:+UseMembar -XX:+UseConcMarkSweepGC -XX:+CMSParallelRemarkEnabled -XX:+CMSScavengeBeforeRemark -XX:ParallelCMSThreads=4 -XX:+UseCMSCompactAtFullCollection -XX:+CMSClassUnloadingEnabled -XX:CMSInitiatingOccupancyFraction=50 -XX:+UseCompressedOops
```

7. 序列化类
`spark.serializer`已经是`kyro`了。

