---
title: Future/FutureTask笔记
date: 2018-08-05 17:32:45
tags: 
- java
- 并发
categories:
- java
- 并发
---

## Future和Callable
`Future`是个接口：
```java
public interface Future<V> {
    boolean cancel(boolean mayInterruptIfRunning);
    boolean isCancelled();
    boolean isDone();
    V get() throws InterruptedException, ExecutionException;
    V get(long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException;
}
```
可见它的功能主要包括：
1. 下取消指令;
2. 查询状态: 取消或者完成;
3. 获取结果。// 这一点和callable一样。

回顾`callable`的源码:
```java
public interface Callable<V> {
    V call() throws Exception;
}
```
可见两者区别主要在于`Future`并不定义任务详情，多了任务执行管理查询接口。
没有`callable`的call函数。

实际使用的时候它们一般的分工如下:
```java
Future<String> future = executorService.submit(new MyCallable());
```

## FutureTask
FutureTask是一个实际的类.

### 外部使用/接口:
```java
Callable<Integer> callable = () -> new Random().nextInt(100);
FutureTask<Integer> future = new FutureTask<>(callable);
// 方法1: 可以由当前线程执行:
// future.run();
// 方法2: 可以由别的线程来执行:
new Thread(future).start();
System.out.println(future.get());
```
可以看出`FutureTask`可以由`Callable`构造，然后用于执行、异步获取结果。



### 内部组件:
源码:
```java
public class FutureTask<V> implements RunnableFuture<V> 
    private volatile int state;// 可能的取值如下: 
    private static final int NEW          = 0; // 初始状态
    private static final int COMPLETING   = 1; // 计算中
    private static final int NORMAL       = 2; // 正常结束 （终结状态）
    private static final int EXCEPTIONAL  = 3; // 异常结束 （终结状态）
    private static final int CANCELLED    = 4; // 已取消   （终结状态）
    private static final int INTERRUPTING = 5; // 打断中
    private static final int INTERRUPTED  = 6; // 已打断   （终结状态）

    /** The underlying callable; nulled out after running */
    private Callable<V> callable;
    /** The result to return or exception to throw from get() */
    private Object outcome; // non-volatile, protected by state reads/writes
    /** The thread running the callable; CASed during run() */
    private volatile Thread runner;
    /** Treiber stack of waiting threads */
    private volatile WaitNode waiters;
// 省略很多方法: ... 
```

其中状态可能的流转如下:（对应四个终结状态的生成）
```
NEW -> COMPLETING -> NORMAL
NEW -> COMPLETING -> EXCEPTIONAL
NEW -> CANCELLED
NEW -> INTERRUPTING -> INTERRUPTED
```
// TODO 为啥异常状态不用负数标示呢？


其中`RunnableFuture`接口是这样的：
```java
public interface RunnableFuture<V> extends Runnable, Future<V> {
    void run();
}
```
可见`FutureTask`是`Runnable`和`Future`的结合, 拥有`Future`的查询结果特性和`Runnable`的定义计算任务的特性。

然而它的内部实现实际上是用了`Callable`而不是`Runnable`:
```java
public FutureTask(Callable<V> callable) {
        if (callable == null)
            throw new NullPointerException();
        this.callable = callable;
        this.state = NEW;       // ensure visibility of callable
    }
public FutureTask(Runnable runnable, V result) {
        this.callable = Executors.callable(runnable, result);
        this.state = NEW;       // ensure visibility of callable
    }    
```
可以看到它只有两个构造函数，即使传入的`Runnable`也会被装配成`Callable`。

### RunnableFuture/FutureTask与AbstractExecutorService
线程池的submit方法默认有如下3个实现: （来自`AbstractExecutorService`类）
```java
public Future<?> submit(Runnable task) {
    if (task == null) throw new NullPointerException();
    RunnableFuture<Void> ftask = newTaskFor(task, null);
    execute(ftask);
    return ftask;
}

public <T> Future<T> submit(Runnable task, T result) {
    if (task == null) throw new NullPointerException();
    RunnableFuture<T> ftask = newTaskFor(task, result);
    execute(ftask);
    return ftask;
}
 
public <T> Future<T> submit(Callable<T> task) {
    if (task == null) throw new NullPointerException();
    RunnableFuture<T> ftask = newTaskFor(task);
    execute(ftask);
    return ftask;
}
```
因此线程池中传入任务后，实际上会转化为`RunnableFuture`.

其中构造`RunnableFuture`的方法如下.
```java
protected <T> RunnableFuture<T> newTaskFor(Runnable runnable, T value) {
    return new FutureTask<T>(runnable, value);
}
protected <T> RunnableFuture<T> newTaskFor(Callable<T> callable) {
    return new FutureTask<T>(callable);
}
```
因此`RunnableFuture`的实际实现，在线程池中默认是`FutureTask`。

综上所述：
因此线程池中传入任务后，实际上会转化为`FutureTask`.
