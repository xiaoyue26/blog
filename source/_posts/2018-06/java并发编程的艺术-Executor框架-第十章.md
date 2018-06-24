---
title: java并发编程的艺术-Executor框架-第十章
date: 2018-06-24 20:32:38
tags: 
- java
- 并发
categories:
- java
- 并发

---


- 两级调度:
```
上层调度: Executor框架控制;
下层调度: 操作系统内核控制。
```

# Executor框架
3大部分:
## 1. 任务
> Runnable/Callable接口。

## 2. 任务的执行
> Executor、ExecutorService。
具体实现包括: ThreadPoolExecutor,ScheduledThreadExecutor...

## 3. 任务的结果(异步计算的结果)
> Future接口/FutureTask类。

*最近太忙了...to be continue...*

