---
title: jvm运行参数配置含义
date: 2018-11-25 20:33:33
tags:
- java
- jvm
categories:
- java
- jvm

---

# 示例1:sparkstreaming的driver配置
```
-Djava.library.path=$JAVA_LIBRARY_PATH:/data/gaiaadmin/gaiaenv/tdwgaia/lib/native
-verbose:gc
-XX:-PrintGCDetails 
-XX:+PrintGCDateStamps
-Xloggc:<LOG_DIR>/gc.log
-XX:MaxPermSize=256m 
-XX:SurvivorRatio=4 
-XX:+UseMembar 
-XX:+UseConcMarkSweepGC
-XX:+CMSParallelRemarkEnabled 
-XX:+CMSScavengeBeforeRemark 
-XX:ParallelCMSThreads=4 
-XX:+UseCMSCompactAtFullCollection 
-XX:+CMSClassUnloadingEnabled 
-XX:CMSInitiatingOccupancyFraction=50 
-XX:+UseCompressedOops
```
含义:
```
-verbose:gc是官方推荐配置,等效于-XX:+PrintGC,表示打印gc日志;
-XX:-PrintGCDetails 减号,关闭详细gc日志;
-XX:+PrintGCDateStamps  加号,打开gc日志日期;
-Xloggc:<LOG_DIR>/gc.log 指定gc日志路径;
-XX:MaxPermSize=256m  最大永生代大小;
-XX:SurvivorRatio=4  两个survivor:新生代的比例;
8的话: 2:8,每个survivor占1/10;
4的话: 2:4,每个survivor占1/6;
-XX:+UseMembar 使用真内存屏障
-XX:+UseConcMarkSweepGC 使用ParNew & CMS（serial old为替补）搜集器
-XX:+CMSParallelRemarkEnabled CMS第二次标记,使用并发标记
-XX:+CMSScavengeBeforeRemark 第二次标记前进行一次minorGC
-XX:ParallelCMSThreads=4 回收线程数
-XX:+UseCMSCompactAtFullCollection 防止堆碎片引起full gc,开启CMS阶段进行合并碎片选项
-XX:+CMSClassUnloadingEnabled 开启回收Perm永生代
-XX:CMSInitiatingOccupancyFraction=50 在老生代占满50%的时候开始回收
-XX:+UseCompressedOops 压缩指针(64位有效)
```

# 示例2:某spring boot进程jvm非默认配置
`jinfo -flags PID`结果:
```
Non-default VM flags: 
-XX:CICompilerCount=15 
-XX:InitialHeapSize=2107637760 
-XX:MaxHeapSize=32210157568 
-XX:MaxNewSize=10736369664 
-XX:MinHeapDeltaBytes=524288 
-XX:NewSize=702545920 
-XX:OldSize=1405091840 
-XX:+UseCompressedClassPointers 
-XX:+UseCompressedOops 
-XX:+UseParallelGC
```
含义:
```
-XX:CICompilerCount=15 JIT的编译线程数
-XX:InitialHeapSize=2107637760  初始堆大小
-XX:MaxHeapSize=32210157568     最大堆大小
-XX:NewSize=702545920        新生代初始大小
-XX:MaxNewSize=10736369664   新生代最大大小
-XX:MinHeapDeltaBytes=524288 每次扩展堆的时候最小增长
-XX:OldSize=1405091840       老生代初始大小
-XX:+UseCompressedClassPointers 压缩指针
-XX:+UseCompressedOops          压缩优化
-XX:+UseParallelGC           Parallel Scavenge + Serial Old 吞吐率优先
```

# jvm配置
参考https://www.cnblogs.com/z-sm/p/6253335.html
## 分类
`-X`: 稳定选项;
`-XX`: 非标准选项;