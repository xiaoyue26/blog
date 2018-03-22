---
title: java并发编程的艺术-第五章
date: 2018-03-21 21:45:20
tags: 
- java
- 并发
categories:
- java
- 并发

---

# 第五章 java中的锁
这章主要写java并发包中与锁相关的`api`.
先是`使用`，然后是`实现`。

# 5.1 Lock接口
`Lock`接口由Java SE 5新增，对飙的是旧java中的`synchronized`内置锁。
主要新增了三个api:
1. 非阻塞获取锁（`tryLock`）;
2. 能被中断地获取锁(`lockInterruptbly`);
3. 超时获取锁(`tryLock(long time,TimeUnit unit)`).
此外还增加了`Condition`接口，增加了等待队列的数量及相关特性。
（`Condition`使用`await()`，对飙的是原来的`obj.wait()`）

示例代码：
```
Lock lock=new ReentrantLock();
lock.lock();// 没获取到锁的话抛异常
try{
    // do something
}
finally{
    lock.unlock();
    // 如果lock.lock()放try里，这里就可能会出错。
    // 因为没获取到锁时，不能unlock。
}
```

|主要Api|备注|
|:-: | -: | 
|lock()| 阻塞式地获取锁。只有在获取到锁后才处理interrupt信息。
|lockInterruptibly()| 获取锁。(可中断)
|tryLock()| 非阻塞地获取锁。不论成败立即返回。
|tryLock(long time,TimeUnit unit)| 超时获取锁，接受中断
|unlock()| 释放锁。
|Condition newCondition()| 获取等待通知组件。


回顾之前的线程状态,lock方法会让线程进入`Blocked`状态。
线程状态与可中断的关系：
1. new: ???
2. Runnable: 不可中断（除非主动检查）
3. Terminated: 不可中断;
4. Blocked: 不可中断; (在`synchronized`时阻塞住不会响应中断)
5. Waiting: 可中断；(sleep方法，wait方法都会抛异常)
6. Time_waiting： 可中断；


简单得说，就只有`Waiting`和`Time_waiting`状态可以中断，或者自己在代码里手动检查中断状态。

**本章的内容有点乱，不太通顺，下面的内容按照个人理解重排。**

# 5.6 Condition接口
`Condition`接口对飙的是原来的`wait/notify`机制，新推出的是`await/signal`机制。
`wait/notify`依赖`synchronized`获得锁，而`Condition`依赖`Lock`获得锁。
原来的`wait/notify`:
```
// A:
synchronize(obj){
    obj.wait();
}
// B:
synchronize(obj){
    obj.notify();
}
```
使用`Condition`的`await/signal`:
```
Lock lock=new ReentrantLock();
Condition condition=lock.newCondition();

// A:
lock.lock();
try{
    condition.await();
}
finally{
    lock.unlock();
}
// B:
lock.lock();
try{
    condition.signal();
}
finally{
    lock.unlock();
}
```

- 功能上：
两种实现都是使用一个对象进行同步操作（加锁解锁）。消费者在获得锁后，在该对象上进行等待（释放锁等待）；生产者获得锁后，通知在该对象上等待的其他线程。
区别在于`synchronized`不需要关心锁的释放，而`Lock`接口需要在finally中手动确保锁的释放。
`wait`只有一个等待队列，而由于一个`Lock`可以生成多个`Condition`，因此`await`可以有多个等待队列。

**案例之有界队列**
```

import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;

public class BoundedQueue<T> {
    private Object[] items;
    private int addIndex, removeIndex, count;
    private Lock lock = new ReentrantLock();
    private Condition notEmpty = lock.newCondition();
    private Condition notFull = lock.newCondition();

    public BoundedQueue(int size) {
        items = new Object[size];
    }

    public void add(T t) throws InterruptedException {
        lock.lock();
        try {
            while (count == items.length) {
                notFull.await();
            }
            items[addIndex] = t;
            if (++addIndex == items.length) { // 循环数组
                addIndex = 0;
            }
            ++count;
            notEmpty.signal();
        } finally {
            lock.unlock();
        }
    }

    public T remove() throws InterruptedException {
        lock.lock();
        try {
            while (count == 0) {
                notEmpty.await();
            }
            Object x = items[removeIndex];
            if (++removeIndex == items.length) {// 循环数组
                removeIndex = 0;
            }
            --count;
            notFull.signal();
            return (T) x;
        } finally {
            lock.unlock();
        }
    }

}
```
上述案例中使用一个锁（`Lock`对象），而有两个等待队列，分别等待`notEmpty`和`notFull`条件。

- 实现逻辑上：（逻辑上的工作流程）
1. 线程尝试获取锁，如果成功则占据锁，失败则进入同步队列；
2. 如果成功获取锁，且进行等待（`wait`或`await`），则进入等待队列，释放锁，通知同步队列中的后继节点；
3. 如果成功获取锁，且进行通知（`notify`或`signal`），则等待队列的首节点挪入同步队列，释放同步锁，通知同步队列中的后继节点。


两个后面会反复用到的概念：
- `同步队列`： 获取同步锁失败后的线程进入同步队列；
- `等待队列`： 获取同步锁成功后，主动等待的线程进入等待队列。

# 5.2 队列同步器
前文中锁的实现中，一个很重要的组件是：
`队列同步器：AQS(AbstractQueuedSynchronizer)`。
此外，其他同步组件的基础框架也是使用AQS。（如`ReentrantLock`,`CountDownLatch`）
回顾55页3.5.4节中的架构层次：
```
// 自顶向下：
1.  Lock,同步器,阻塞队列,Executor,并发容器
2.  AQS,非阻塞数据结构,原子变量类
3.  volatile读写,CAS操作
```


AQS是一个抽象类。
AQS中自底向上的3类方法：
1. 基础方法。(固定)
2. 可重写的方法（调用1的方法）；
3. 模版方法；（固定，调用1，2的方法）

AQS在设计上是基于模版方法模式的抽象类。也就是说，我们需要新增一个子类继承AQS，然后重写上述第二类方法。

**基础方法**
1. `getState()`: 获取当前同步状态；
2. `setState()`: 设置当前同步状态；
3. `compareAndSetState(int expect,int update)`: 使用CAS设置当前状态,该方法保证设置的原子性。


**可重写的方法**

|可重写的方法|描述|
|:-: | -: | 
|boolean tryAcquire(int arg)| 独占式获取同步状态。（调CAS）
|boolean tryRelease(int arg)| 独占式释放同步状态。
|int tryAcquireShared(int arg)| 共享式获取同步状态。返回值>=0则成功。
|boolean tryReleaseShared(int arg)| 共享式释放同步状态。 
|boolean isHeldExclusively(int arg)| 判断是否被当前线程独占。
 
 
**模版方法**

|模版方法|描述|
|:-: | -: | 
|void acquire(int arg)| 独占式获取同步状态（调用上面的`tryAcquire`），失败则进入同步队列。
|void acquireShared(int arg)| 共享式获取同步状态，失败则进入同步队列。
|boolean release(int arg)| 独占式释放同步状态
|boolean releaseShared(int arg)| 共享式释放同步状态
|Condition< Thread>getQueuedThreads()|获取同步队列线程集合
|其他| 其他响应中断/超时返回的版本的方法









-- to be continue
