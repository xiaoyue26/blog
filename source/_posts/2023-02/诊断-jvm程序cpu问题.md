---
title: 诊断-jvm程序cpu问题
date: 2023-02-01 11:06:43
tags: 
- java
- jvm
- jfr

categories:
- java
- 性能

---

# 摘要
{% img /images/2023-02/abstract.png 800 1200 abstract %}

# TOP命令
进程出现cpu问题的时候，最常见的思路就是用top命令看一下是哪个线程导致的:
```shell script
# 只看rpc进程的各线程cpu分布:(可以修改RpcServerModularStarter部分来锁定自己的进程)
nohup top -b -d 2 -n 2592000 -H -p $(pgrep -d',' -f RpcServerModularStarter) | grep 'top ' -A 30  >> top.log 2>&1 &
echo $! > nohup2.pid

# 看机器上所有进程的cpu分布:
nohup top -bc -d 2 -n 2592000 >> top.log 2>&1 &
echo $! > nohup.pid

tail -fn 100 top.log
```
比如以上命令可以每隔2秒打印最高cpu占用的进程、线程到日志中。
占用内存、磁盘空间不大，可以长期开启，但是需要手动，比较麻烦。

# 容器云监控、诊断
公司有监控系统，自动开启了许多监控，我们可以看中间件的情况、cpu/线程池的历史情况：
信息安全因素这里略。
本质就是定时调用各个线程池的：
```java
java.util.concurrent.ThreadPoolExecutor.getTaskCount()
java.util.concurrent.ThreadPoolExecutor.getActiveCount()
```

# async-profiler
配合jfr命令或者JDK Mission Control，可以定位常见的cpu,lock,alloc相关事件的问题。
参考之前写的: https://xiaoyue26.github.io/2022/03/25/2022-03/G1%E8%B0%83%E4%BC%98-%E5%A4%8D%E6%9D%82%E4%B8%9A%E5%8A%A1%E6%B2%BB%E7%90%86%E5%B0%8F%E8%AE%B0/

（-e参数可以指定cpu,lock,alloc，分别针对不同的jfr事件，可以生成.jfr文件）

参考：https://github.com/jvm-profiling-tools/async-profiler

`jmc(JDK Mission Control)`: https://adoptium.net/jmc/

# JFR监控
JFR: Java Flight Record （Java飞行记录)
JVM内置的黑匣子。jdk11中支持136个事件，
前面的async-profiler中生成的jfr只会记录其中少量一些cpu\lock\alloc相关的事件，所以优点生成的jfr文件会比较小。
缺点就是有些信息不够详细，也不够定制化。
JFR作为jvm内置的黑匣子，支持的事件非常细而全，目标是定位jvm所有问题:
{% img /images/2023-02/jfr-events.png 800 1200 events %}
（来自：https://bestsolution-at.github.io/jfr-doc/
，这个网站里还有各个event的简要信息）
{% img /images/2023-02/event-type.png 800 1200 event-type %}
{% img /images/2023-02/jfr-buffer.png 800 1200 jfr-buffer %}

使用流程:
1。定制jfc文件(配置需要打印的events);
2。用jfc采集jfc。

## 1.定制化event事件(生成jfc)
{% img /images/2023-02/jfc-init.png 800 1200 jfc-init %}
{% img /images/2023-02/jfc-edit.png 800 1200 jfc-edit %}

## 2.采集事件(生成jfr)
```shell script
# 开始监控:
jcmd <pid> JFR.start name=jfr_profile filename=/data/logs/8090/res.jfr maxage=1h maxsize=1g disk=true \
settings=/data/logs/8090/config.jfc

# 结束监控:
jcmd <pid> JFR.stop name=jfr_profile
```
生成res.jfr文件以后，可以用`JDK Mission Control`打开进行图形化的分析：

### 内存分配
可以按thread,class,方法聚合统计
{% img /images/2023-02/jfr-alloc.png 800 1200 jfr-alloc %}
{% img /images/2023-02/jfr-alloc2.png 800 1200 jfr-alloc2 %}
### 锁
可以按class,address聚合统计;
{% img /images/2023-02/jfr-lock.png 800 1200 jfr-lock %}
如果cpu问题不是某几个线程cpu占用特别高导致的，一般就是"惊群效应"导致的。
也就是一大堆线程同时被唤醒（每个线程只占一点点cpu，但是累积起来就很多）。
而这种情况往往就是这些线程同时在等待某个锁，所以cpu问题看锁的统计是很有用的。

### code dump
可以查看挂掉时候的调用栈信息:
{% img /images/2023-02/jfr-dump.png 800 1200 jfr-dump %}

# 总结
cpu问题可以分为两类：
1。某几个线程占大头：用top监控，可以配合jstack找出即可；
可以参考http://xiaoyue26.github.io/2022/11/21/2022-11/cpu%E6%AF%9B%E5%88%BA%E9%97%AE%E9%A2%98%E9%80%9A%E7%94%A8%E4%BC%98%E5%8C%96%E6%80%9D%E8%B7%AF/

2。没有特别高占用的线程，每个都占很少（比如3%）累积起来很多造成突刺:
惊群效应；可以找会阻塞住很多线程的方向：classLoader/jit/锁监控。

# 参考资料
https://www.zhihu.com/column/c_1264859821121355776
