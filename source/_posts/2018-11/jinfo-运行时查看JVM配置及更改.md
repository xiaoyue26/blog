---
title: jinfo-运行时查看JVM配置及更改
date: 2018-11-25 20:12:57
tags:
- java
- jvm
categories:
- java
- jvm

---

# 查看运行的java程序的启动命令
`jps -lvm`

# 运行时查看JVM配置
语法如下:
`jinfo -flag <name> PID`
如:
`jinfo -flag CMSAbortablePrecleanWaitMillis 11715`

查看全部非默认配置:
`jinfo -flags PID`
或者:
`jcmd PID VM.flags`

# 运行时修改JVM配置
首先并不是所有JVM配置都能在运行时动态修改的,需要查看哪些配置是可以运行时动态修改的:
## 列出可运行时调整的JVM配置(linux)
`java -XX:+PrintFlagsInitial | grep manageable`
结果如下:
```
intx CMSAbortablePrecleanWaitMillis = 100 {manageable}
intx CMSTriggerInterval  = -1 {manageable}
intx CMSWaitDuration = 2000 {manageable}
bool HeapDumpAfterFullGC = false {manageable}
bool HeapDumpBeforeFullGC   = false {manageable}
bool HeapDumpOnOutOfMemoryError  = false {manageable}
ccstr HeapDumpPath = {manageable}
uintx MaxHeapFreeRatio = 70 {manageable}
uintx MinHeapFreeRatio = 40 {manageable}
bool PrintClassHistogram = false {manageable}
bool PrintClassHistogramAfterFullGC = false {manageable}
bool PrintClassHistogramBeforeFullGC  = false {manageable}
bool PrintConcurrentLocks   = false {manageable}
bool PrintGC  = false {manageable}
bool PrintGCDateStamps   = false {manageable}
bool PrintGCDetails   = false {manageable}
bool PrintGCID = false {manageable}
bool PrintGCTimeStamps   = false {manageable}
```

可以看到参数的类型主要分为`bool`和非`bool`(`intx`,`uintx`,`ccstr`)。
## 运行时修改bool型配置
用加减(+/-)表示开关:
`jinfo -flag [+|-]<name> PID`

## 运行时修改intx/ccstr/uintx型配置
` jinfo -flag <name>=<value> PID`


# 示例
比如我们有一个java进程11715,我们突然想查看它的gc情况,但是之前又没有打开gc日志,只重定向了控制台日志,这个时候可以动态打开:
```
jinfo -flag +PrintGC 11715
jinfo -flag +PrintGCDateStamps 11715
jinfo -flag +PrintGCDetails 11715
```
拿到日志:
```
2018-11-25T20:21:41.772+0800: [GC (Allocation Failure) [PSYoungGen: 45952K->1856K(48128K)] 248269K->204197K(309248K), 0.0423007 secs] [Times: user=0.88 sys=0.02, real=0.06 secs]
```

调试完后再关掉即可:
```
jinfo -flag -PrintGC 11715
jinfo -flag -PrintGCDateStamps 11715
jinfo -flag -PrintGCDetails 11715
```



