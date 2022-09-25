---
title: 调优-解决线程池造成的进程卡顿、cpu毛刺问题
date: 2022-09-25 15:11:32
tags: 
- java
- 线程池
categories:
- java
- 性能

---

# 背景&现象
## 背景
有个qps较高的rpc，使用了线程池来并发请求多个下游服务，且设定了对于下游的超时时间。

## 现象
rpc服务调用下游时，时不时出现耗时很高的情况，平均耗时挺低的，但是p99和p995就比较高了，达到秒级，超过了设定的调用下游的超时时间。
从rpc monitor上看，调用下游实际也用不了这么长时间，说明不是跨span的耗时；
使用arthas  trace查看方法级的耗时:
{% img /images/2022-09/trace_supply.png 800 1200 trace_supply %}
{% img /images/2022-09/start.png 800 1200 start %}
可以发现是频繁start新的thread导致的。

再查看日志里的线程编号：
-thread-7665821
已经达到七百万了！


参考以前的学习：
http://xiaoyue26.github.io/2022/03/14/2022-03/%E4%B8%AD%E6%96%AD%E6%A2%B3%E7%90%86/
{% img /images/2022-09/core_thread.png 800 1200 core_thread %}
由于jvm线程与内核线程一一对应(hotpot jvm下)，频繁回收、创建带来的内核态切换、系统调用开销过于大了。


# 解决方案
综上可以发现线程池里有频繁的线程回收、再创建操作，需要关闭core线程自动回收的机制（基础框架内默认是开启回收）
```java
DynamicThreadExecutor.dynamic(POOL_SIZE::get, "csc-center-executor-%d", false);
```

字段注释：
```java
@param allowCoreThreadTimeout 是否允许核心线程超时后被回收，
*         超时时间参数可通过为{@link ExecutorBuilder.coreThreadKeepAliveMillis}设置
*         默认超时时间为1分钟
```

# 收益
请求耗时p99\p995、超时异常大幅降低；
{% img /images/2022-09/p99.png 800 1200 p99 %}
{% img /images/2022-09/timeout.png 800 1200 timeout %}

# 更多参考代码
也可以自己选择喜欢的builder创建:
```java
private final DynamicThreadExecutor fromBuilder = DynamicThreadExecutor.dynamic(POOL_SIZE::get,
       num -> ExecutorsEx.newBlockingThreadPool(ExecutorBuilder
               .newBuilder()
               .allowCoreThreadTimeout(false)
               .threadSize(num)
               .queueSize(num)
               .theadNameFormat("from-builder-%d")));
```

# 注意事项(可能的坑)
1. 摘要中的代码默认创建的是BlockingThreadPool，不支持嵌套使用。不要在提交的任务中再次提交任务到同一个线程池，以免死锁。
(jdk中的支持嵌套调用)


# 参考资料
https://zhuanlan.zhihu.com/p/342929293

