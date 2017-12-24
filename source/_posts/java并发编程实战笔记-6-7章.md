---
title: java并发编程实战笔记-6-7章
date: 2017-02-25 19:45:43
tags: java
categories:
- java
- 并发
---

# 6.1节 背景
(为了引出Executor框架)
任务执行方式:
1. 串行; 单线程,太慢;
2. 完全并行; 每个任务一个线程,开销太大.
3. 使用Executor框架. OK 

# 6.2 Executor框架
`java.util.concurrent`提供的线程池.
可以通过实现Executor接口,自定义执行策略:
1. 谁来执行;
2. 执行顺序;(FIFO,LIFO,优先级)
3. 并发度;
4. 线程池容量(包括等待的);
5. 什么时候拒绝任务,拒绝哪一个;
6. 执行任务前后的操作.

或者直接使用`Executors`中编写好的线程池/执行策略:
```
newFixedThreadPool // 定长
newCachedThreadPool // 无限增长,但会复用原来的
newSingleThreadExecutor // 单线程
newScheduledThreadPool // 定长,但可以定时\延迟执行.
newWorkStealingPool // 用fork-join的,工作觅取的 // 1.8新增.
```

## Executor的生命周期
为了以各种方式关掉`Executor`,库中写了`ExecutorService`:
```
public interface ExecutorService extends Executor {
        void shutdown();//平缓得关闭,不再接收新任务,执行剩余的;
        List<Runnable> shutdownNow();//取消运行中的和未执行的;
        boolean isShutdown();//是否已经下达shutdown命令
        boolean isTerminated();//是否完成了shutdown命令
        boolean awaitTermination(long timeout, TimeUnit unit)
                throws InterruptedException;
        // ... 还有一些invoke和submit        
    }
```

ExecutorService生命周期有3种状态:
运行,关闭,已终止. 

ExecutorService中的任务的生命周期:
创建,提交,开始,完成.

上述生命周期都是单向不可逆的.

# 线程池的局限
1. 适用于同构任务,异构任务分解粒度不够细,提升不够大;

# 取消与关闭
## 背景:
> java中无法简单\安全得停止取消某个线程;
需要使用中断(一种协作机制),从一个线程发出取消请求,中断另一个线程.因此其实需要被中断的线程预先提供安全停止\取消的方法,其中包括清理资源等操作.

## 取消策略:
1. HOW: 其他线程如何请求取消;
2. WHEN: 本线程何时受理取消请求;
3. WHAT: 取消时具体要干什么.

Thread中的中断方法:
```
public class Thread{
public void interrupt(){}// 中断此线程
public boolean isInterrupted(){}//查询中断状态
public static boolean interrupted(){}// 查询中断状态,且清除中断.
}
```

## 中断策略
有些线程不支持取消,但可以支持中断. 

中断的方法:
1. 直接中断:
```
Thread.currentThread().interrupt();
```
2. 限时任务:
``` 
Future<?>task=exec.submit(r);
task.get(timeout,unit);
task.cancel();
```

3. 处理不可中断的阻塞:

(1)java.io包中的同步Socket I/O:
`InputStream`和`OutputStream`的read,write方法都不会响应中断.
中断方法: 关闭底层套接字,让read,write抛出SocketException. 

(2)java.io包中的同步I/O:
中断方法:
中断`InterruptibleChannel`.抛出`ClosedByInterruptExeception`.
关闭`InterruptibleChannel`.抛出`AsynchronousCloseException`.


(3)Selector的异步I/O:(java.nio.channels)
中断方法:
close或wakeup方法. 抛出`ClosedSelectorException`.

(4)等待内置锁.
使用Lock类中的`lockInterruptible`方法.
