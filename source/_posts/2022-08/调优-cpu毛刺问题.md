---
title: 调优-cpu毛刺问题
date: 2022-08-30 18:07:52
tags: 
- java
- caffeine
- 线程池
categories:
- java
- 性能

---

# 摘要
1。线上服务追踪使用公共线程池的调用栈；
2。动态调整线程池大小;
3。拓展：pstack、strace非java进程;


# 背景
线上服务偶尔会有一两个实例突然cpu飙到100%，尤其以刚启动的时候发生的概率高。
虽然分钟级的平均使用率只有35%左右，但是秒级则会有几秒进入进程卡顿状态，影响服务的可用性。

# 问题定位
## 1。定位线程池:
cpu问题，首先想到的是线程池打满的可能。所以首先看监控里各个线程池的使用率。
然后发现打满的线程池名字是: `fork-join-common-pool`。

这里就很尴尬了，因为如果是业务命名好的线程池，就可以立即知道涉及到的业务和相关代码的位置了。

> 这里监控线程池使用率用的是java.util.concurrent.ThreadPoolExecutor.getActiveCount
> 或java.util.concurrent.ForkJoinPool.getActiveThreadCount
> 对比java.util.concurrent.ThreadPoolExecutor.getMaximumPoolSize
> 或java.util.concurrent.ForkJoinPool.getParallelism
> 启动所有线程池的时候注册一下reporter即可。

## 2。追踪调用栈
这种不命名线程池的不规范的使用，给定位问题带来了麻烦。
需要登陆到容器里，然后用`arthas`连上jvm:
```shell script
stack java.util.concurrent.ForkJoinPool externalSubmit -n 3
# 需要设置options unsafe true
```

看一下提交任务的代码的调用栈:
```
ts=2022-08-24 18:16:20;thread_name=csc-infra-executor-348;id=f2e45;is_daemon=false;priority=5;TCCL=jdk.internal.loader.ClassLoaders$AppClassLoader@531d72ca
    @java.util.concurrent.ForkJoinPool.signalWork()
        at java.util.concurrent.ForkJoinPool.externalPush(ForkJoinPool.java:1903)
        at java.util.concurrent.ForkJoinPool.externalSubmit(ForkJoinPool.java:1921)
        at java.util.concurrent.ForkJoinPool.execute(ForkJoinPool.java:2453)
        at com.github.benmanes.caffeine.cache.BoundedLocalCache.scheduleDrainBuffers(BoundedLocalCache.java:1427)
        at com.github.benmanes.caffeine.cache.BoundedLocalCache.scheduleAfterWrite(BoundedLocalCache.java:1394)
        at com.github.benmanes.caffeine.cache.BoundedLocalCache.afterWrite(BoundedLocalCache.java:1364)
        at com.github.benmanes.caffeine.cache.BoundedLocalCache.doComputeIfAbsent(BoundedLocalCache.java:2470)
        at com.github.benmanes.caffeine.cache.BoundedLocalCache.computeIfAbsent(BoundedLocalCache.java:2386)
        at com.github.benmanes.caffeine.cache.LocalCache.computeIfAbsent(LocalCache.java:108)
        at com.github.benmanes.caffeine.cache.LocalLoadingCache.get(LocalLoadingCache.java:54)
        at com.github.benmanes.caffeine.guava.CaffeinatedGuavaLoadingCache.get(CaffeinatedGuavaLoadingCache.java:59)
        at 
```
结果挺意外的，原来是使用的`caffeine`的缓存默认用的公共线程池。
翻了一下公司封装的接口里没法往里面传自己的线程池，不传默认就用公共forkjoin线程池，worker数量=core * 2;
因此吞吐量太低了。一旦遇到有大量缓存需要重载的时候，就会卡住，因此会cpu毛刺。
平时因为设置得过期时间和刷新时间好，不会触发；启动的时候，缓存还是空的，因此线程池会打满，请求堆积、cpu毛刺。


## 3。解决方案
首先是推动公司基础部门把这个漏暴露的接口补上；
其次是使用自己创建的线程池时，要确定线程池的大小。
这个问题一方面有网上的公式:

> cpu密集: 线程池大小=core * 2;
> IO密集: 线程池大小= IO时间/cpu时间 * 2;

其实一般业务系统，又不是机器学习之类的算法程序，基本上都是IO密集；
可以算一下下游返回时间大概多少(超时时间)，除以(总时间-IO时间)即可。

除了根据上面的公式直接拍，如果后续下游有变化，动态调整也很重要，所以最好是用可以动态调整线程池大小的封装。
可以借用`com.google.common.util.concurrent.ForwardingExecutorService`，稍微封装一下。
主要注意的就是调大`corePoolSize`之前，要先调大`maximumPoolSize`; 
反之则反过来。（也不是每种线程池都能调,jdk默认是让调的）


## 4。拓展
jvm进程我们一般先:
1.top一下看进程id;
2.`top -Hp <pid>`看进程内线程id(十进制);
3.`printf "%x\n" <pid>`看16进制nid;
4.`jstack <pid> | grep <nid> -A 30`看具体栈;

如果是非jvm程序，则不能用`jstack`,可以用`pstack`:
```shell script
pstack <线程id>
``` 
不过信息就比jstack少很多了。
还可以:
```shell script
strace -o strace.log -tt -p <线程id>
```
追踪运行时的汇编。







