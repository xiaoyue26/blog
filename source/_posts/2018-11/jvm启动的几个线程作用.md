---
title: jvm启动的几个线程作用
date: 2018-11-27 12:01:16
tags:
- java
- jvm
categories:
- java
- jvm

---

# jvm启动的几个线程作用:
1. `Signal Dispatcher`: 接受各种信号;
2. `Attach Listener`: 监听各种请求的socket连接,把执行的操作扔给`VM Thread`。
这里的请求包括:
```java
static AttachOperationFunctionInfo funcs[] = {
  { "agentProperties",  get_agent_properties },
  { "datadump",         data_dump },
  { "dumpheap",         dump_heap },
  { "load",             JvmtiExport::load_agent_library },
  { "properties",       get_system_properties },
  { "threaddump",       thread_dump },
  { "inspectheap",      heap_inspection },
  { "setflag",          set_flag },
  { "printflag",        print_flag },
  { "jcmd",             jcmd },
  { NULL,               NULL }
};
```
3. `VM Thread`: 线程母体,最原始的线程,单例,里面有个队列，存放上面的操作,它负责loop处理队列中的操作.（包括对其他线程的创建，分配和对象的清理，cms-mark等工作）
4. `CompilerThread0`: JIT动态编译线程;
5. `ConcurrentMark-SweepGCThread`: CMS垃圾收集线程;
6. `DestroyJavaVM`: 负责卸载JVM的线程;
7. `ContainerBackground Processor`: JBoss守护线程;
8. `Dispatcher-Thread-3`: Log4j异步日志守护线程;
9. `Finalizer`: 垃圾回收守护线程;
10. `Gang worker#0`: 新生代回收线程;
11. `GC Daemon`: RMI远程GC线程(调用`system.gc()`);
12. `Low MemoryDetector	`: 发现可用内存低，则分配新的内存空间;
13. `process reaper`: 执行os命令;
14. `Reference Handler`: 处理引用对象本身（软、弱、虚引用）的垃圾回收问题;
15. `SurrogateLockerThread`: CMS垃圾回收;
16. `VM Periodic Task Thread`: 定期的内存监控、JVM运行状况监控。

参考: http://ifeve.com/jvm-thread/