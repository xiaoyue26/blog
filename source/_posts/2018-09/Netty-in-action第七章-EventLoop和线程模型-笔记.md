---
title: Netty in action第七章-EventLoop和线程模型-笔记
date: 2018-09-15 20:34:37
tags: 
- java
- netty
categories:
- java
- netty

---

# 线程模型
## Executor API(java 5)
{% img /images/2018-09/7.1.jpg 400 600 线程池 %}
java 5引入的Executor的执行逻辑是：池化线程，做成一个线程池，复用线程。

优点:
消除了创建和销毁线程的开销；

缺点: 
高负载下上下文切换开销大。

## 事件循环： EventLoop接口
事件循环如下。所谓事件循环就是一个死循环,1次批量处理所有ready的事件，而不是像之前Executor接口那样每个任务切换一次。可以看出高负载的时候比较有优势，低负载的时候有cpu空转。
```java
 public static void executeTaskInEventLoop() {
        boolean terminated = true;
        //...
        while (!terminated) {
            //阻塞，直到有事件已经就绪可被运行
            List<Runnable> readyEvents = blockUntilEventsReady();
            for (Runnable ev: readyEvents) {
                //循环遍历，并处理所有的事件
                ev.run();
            }
        }
    }
```
看代码可以看出一个事件循环(`EventLoop`)就是某一个线程在那死循环执行任务。
所以如果要有多个线程干活，就要多个`EventLoop`,就组成了`EventLoopGroup`。(可以理解成一种批处理任务的线程池)。 看类层次也能发现，`EventLoopGroup`继承自`AbstractExecutorService`。
相应的类层次如下:
{% img /images/2018-09/7.2.jpg 400 600 EventLoop类图 %}

### 执行顺序
事件/任务执行顺序是FIFO的。

## Netty IO和事件处理
**Netty3:**
> 木有EventLoop.
入站事件: IO线程中处理;// 类似于EventLoop
出站事件: 调用线程处理;

缺点:
上下文切换开销;
同步困难(例如当出站事件触发入站事件)。

**Netty4:**
>用EventLoop;
给定EvenLoop，一个线程处理所有事件。
优点:
无需同步（除非是Sharable）,上下文开销降低。


# 任务调度(外部接口)
有两种方式：
1. JDK API;
2. Netty API.

## 1. 使用JDK API
使用`java.util.concurrent.Executors`包里的api。
如果要设定一个60s后执行的任务:
```java
public static void schedule() {
    // 线程池:
    ScheduledExecutorService executor =
            Executors.newScheduledThreadPool(10);
    ScheduledFuture<?> future = executor.schedule(
        new Runnable() {
        @Override
        public void run() {
            System.out.println("Now it is 60 seconds later");
        }
    }, 60, TimeUnit.SECONDS); 
    executor.shutdown();
}
```
缺点:
高负载下性能不足。

## 使用EventLoop调度任务
如果要设定一个60s后执行的任务:
```java
public static void scheduleViaEventLoop() {
    Channel ch = CHANNEL_FROM_SOMEWHERE; // get reference from somewhere
    ScheduledFuture<?> future = ch.eventLoop().schedule(
        new Runnable() {
        @Override
        public void run() {
            System.out.println("60 seconds later");
        }
    }, 60, TimeUnit.SECONDS); // 60s后
}
```
由于`EventLoop`是`ScheduledExecutorService`的子接口，因此有它一样的对外接口。
使用起来没啥区别。

如果要设定一个每隔60s运行的任务:
```java
public static void scheduleFixedViaEventLoop() {
    Channel ch = CHANNEL_FROM_SOMEWHERE; // get reference from somewhere
    ScheduledFuture<?> future = ch.eventLoop().scheduleAtFixedRate(
       new Runnable() {
       @Override
       public void run() {
            System.out.println("Run every 60 seconds");
       }
    }, 60, 60, TimeUnit.SECONDS);
}
```

# 实现细节(内部细节)
## 线程管理
确定当前执行的线程是否是分配给当前Channel的线程。
// 一个Channel的所有事件会在同一个EventLoop里头。

```java
if(当前线程 in 匹配EventLoop中的线程) 直接执行;
else{
    任务放入内部队列。
}
```

{% img /images/2018-09/7.3.jpg 400 600 EventLoop执行逻辑 %}

**优点:**
(1) 避免了线程同步。(只会执行当前Channel的任务)
(2) 减少了上下文切换。(有时候能够不进任务队列，直接执行)

**缺点:**
单个事件可能阻塞整个`Channel`:
由于一个Channel的所有事件都由同一个EventLoop处理，因此要求EventLoop不能阻塞(不然其他事件就无法处理了)。

**解决方案**
长时间运行或阻塞的任务=>创建一个`EventExecutor`来消费掉，以便从`EventLoop`中移除，Netty提供了一个默认的`DefaultEventExecutorGroup`来支持这种操作。把它add进pipeline即可。
// TODO

## EventLoop/线程分配
一个Channel的事件由同一个EventLoop处理。Channel和EventLoop的对应关系由`EventLoopGroup`分配。

## 异步传输
**特点:** 
EventLoop数量少于Channel数量。// 多个Channel共享一个EventLoop。
EventLoop数量固定，不新增。
{% img /images/2018-09/7.4.jpg 400 600 异步EventLoop %}

因为异步，所以少量线程数就能支撑。
具体分配Channel的时候，用`round-robin`实现即可，比较粗糙的负载均衡。

**限制**
由于1个EventLoop用于多个Channel, 意味着`Threadlocal`对象会被多个Channel共享。
需要注意这一点。

## 同步传输
**特点**
EventLoop数量等于Channel数量。 // 不共享EventLoop
EventLoop数量不断增加，每次新增Channel，就新增EventLoop。
{% img /images/2018-09/7.5.jpg 400 600 同步EventLoop %}
