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

## 不同线程池适用场景
- SingleThreadExecutor
单线程，适用于需要保证顺序执行，而且只允许单线程执行的场景；
- CachedThreadExecutor
大小无界，适用于负载较轻，或执行很多短期异步小任务；
- FixedThreadExecutor
大小固定，适用于负载较重的服务器。

## 周期线程池
- ScheduledThreadPoolExecutor
大小固定；
- SingleThreadScheduledExetor
单线程。


## Future接口
submit任务到Executor后，返回一个Future接口对象，目前jdk的实现是FutureTask类对象，以后不一定。
```java
<T>Future<T>submit(Callable<T>task)
```

# ThreadPoolExecutor
四大组件:
1. corePool: 核心线程池大小
2. maximumPool: 最大线程池大小
3. BlockingQueue: 暂时保存任务的工作队列
4. RejectedExecutionHandler: 线程池已经关闭或饱和(达到最大线程池大小且工作队列已满),execute方法调用的Handler。

线程池通用的工作流程:
```
1. 预热: 接到任务就新建线程，直到达到corePoolSize;
2. 正式工作: 接到任务先扔到BlockingQueue，核心线程池的线程不停从BlockingQueue中取任务执行; 
3. 扩容: Blocking满,扩大corePool到maximunPool.
4. 饱和: Blocking满,且达到maximumPool,调用rejectedExecutorHandler.
```

## 创建线程池的方法:
1. 使用`ExecutorService`中提供的定制好的线程池:
```java
Executors.newFixedThreadPool(1);
Executors.newSingleThreadExecutor();
Executors.newCachedThreadPool();
ExecutorService es = Executors.newCachedThreadPool();
```
2. 自己用`ThreadPoolExecutor`定制一个线程池:
```java
// 手动自定义详细参数:
ExecutorService es2 = new ThreadPoolExecutor(10, 10
, 101, TimeUnit.SECONDS
, new LinkedBlockingQueue<Runnable>(2000));
```

其中的继承关系是:
```java
ThreadPoolExecutor->AbstractExecutorService->ExecutorService
ThreadPoolExecutor extends AbstractExecutorService
abstract class AbstractExecutorService implements ExecutorService
```



## FixedThreadPool
```java
// ThreadPoolExecutor实现类中的通用方法:
public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue) {
        this(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue,
             Executors.defaultThreadFactory(), defaultHandler);
    }
//  Executors工具类中的FixedThreadPool:
public static ExecutorService newFixedThreadPool(int nThread){
    return new ThreadPoolExecutor(nThread,nThread,0L,TimeUnit.MILLISECENDS
    ,new LinkedBlockingQueue<Runnable>());
}
```
可以看出最大池和核心池都是n,也就是固定大小，不再扩容。
keekAlive时间为0，因此多余空闲线程立刻被终止。
最后一个参数用的是无界队列，因此没有饱和状态，只有shutdown状态。

## SingleThreadExecutor
```java
public static ExecutorService newSingleThreadExecutor() {
        return new FinalizableDelegatedExecutorService
            (new ThreadPoolExecutor(1, 1,
                                    0L, TimeUnit.MILLISECONDS,
                                    new LinkedBlockingQueue<Runnable>()));
    }
```

## CachedThreadPool
```java
public static ExecutorService newCachedThreadPool() {
        return new ThreadPoolExecutor(0, Integer.MAX_VALUE
            ,60L, TimeUnit.SECONDS
            ,new SynchronousQueue<Runnable>());
    }
```
SynchronousQueue: 同步传递任务的阻塞队列。(容量为0,就是个传球手,总是满状态)

上述三个实现都没有饱和状态，前两者是因为BlockingQueue无界，
CachedThreadPool是因为maximumPool太大了(Integer.MAX_VALUE)。
*最近太忙了...to be continue...*

