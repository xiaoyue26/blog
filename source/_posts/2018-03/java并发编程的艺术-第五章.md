---
title: java并发编程的艺术-第五章(1)
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
```java
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
```java
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
```java
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
```java

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

AQS在设计上是基于模版方法模式的抽象类。也就是说，我们需要新增一个子类继承AQS，然后重写上述第2类方法。而第1类和第3类方法，要么是private的无法继承，要么是final的无法重写。而第二类方法，如果没有重写，默认实现只有一行`throw new UnsupportedOperationException();`,调用的时候就会直接抛异常了。

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

**案例之-独占锁**
`Mutex`的实现：
```java

import java.util.concurrent.TimeUnit;
import java.util.concurrent.locks.AbstractQueuedSynchronizer;
import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.Lock;

public class Mutex implements Lock {
    private static class Sync extends AbstractQueuedSynchronizer {
        @Override
        protected boolean isHeldExclusively() {
            return getState() == 1;
        }

        @Override
        public boolean tryAcquire(int acquireds) {
            if (compareAndSetState(0, 1)) {
                setExclusiveOwnerThread(Thread.currentThread());
                return true;
            }
            return false;
        }

        @Override
        protected boolean tryRelease(int releases) {
            if (getState() == 0) {
                throw new IllegalMonitorStateException();
            }
            setExclusiveOwnerThread(null);
            setState(0);
            return true;
        }

        Condition newCondition() {
            return new ConditionObject();
        }
    }

    // 将操作委托给sync实现即可。
    private final Sync sync = new Sync(); // (代理)

    @Override
    public void lock() {
        sync.acquire(1);
    }

    @Override
    public void lockInterruptibly() throws InterruptedException {
        sync.acquireInterruptibly(1);
    }

    @Override
    public boolean tryLock() {
        return sync.tryAcquire(1);
    }

    @Override
    public boolean tryLock(long time, TimeUnit unit) throws InterruptedException {
        return sync.tryAcquireNanos(1, unit.toNanos(time));
    }

    @Override
    public void unlock() {
        sync.release(1);
    }

    @Override
    public Condition newCondition() {
        return sync.newCondition();
    }

    // 不在Lock接口中，但是有用的方法：
    public boolean isLocked() {
        return sync.isHeldExclusively();
    }

    public boolean hasQueuedThread() {
        return sync.hasQueuedThreads();
    }
}
```
上述`sync`只实现了独占操作，因此调用共享操作会抛异常。
因此`mutex`中只调用了`sync`的独占模版方法。
因此`mutex`最多是一个独占锁。

**案例之-TwinsLock**
用AQS实现一个最多能被两个线程同时占据的锁。
`TwinsLock`实现：
```java
public class TwinsLock implements Lock {

    private static final class Sync extends AbstractQueuedSynchronizer {
        Sync(int count) {
            if (count <= 0) {
                throw new IllegalArgumentException("count must larger than 0");
            }
            setState(count);
        }

        @Override
        public int tryAcquireShared(int reduceCount) {
            for (; ; ) {
                int current = getState();
                int newCount = current - reduceCount;
                if (newCount < 0 || compareAndSetState(current, newCount)) {
                    return newCount;
                }
            }
        }

        @Override
        public boolean tryReleaseShared(int returnCount) {
            for (; ; ) {
                int current = getState();
                int newCount = current + returnCount;
                if (compareAndSetState(current, newCount)) {
                    return true;
                }
            }
        }

        // 额外的:
        Condition newCondition() {
            return new ConditionObject();
        }
    }

    private final Sync sync = new Sync(2);// 最多俩人共享

    @Override
    public void lock() {
        sync.acquireShared(1);
    }

    @Override
    public void unlock() {
        sync.releaseShared(1);
    }

    @Override
    public void lockInterruptibly() throws InterruptedException {
        sync.acquireSharedInterruptibly(1);
    }

    @Override
    public boolean tryLock() {
        return sync.tryAcquireShared(1) >= 0;
    }

    @Override
    public boolean tryLock(long time, TimeUnit unit) throws InterruptedException {
        return sync.tryAcquireSharedNanos(1, unit.toNanos(time));
    }


    @Override
    public Condition newCondition() {
        return sync.newCondition();
    }

}
```
与`Mutex`不同的是，`TwinsLock`中的同步器主要重写了AQS的共享方法，`Lock`接口中也调用的是共享方法。也就是按照共享式访问编写，然后用`count`控制同步资源数。

### 5.2.2 AQS的实现分析
上面讲了`Lock`,`Condition`的使用，`AQS`的使用（用来自定义`Lock`），下面讲`AQS`的实现。

**同步队列**

- 同步队列与等待队列
AQS即队列同步器，很重要的一个概念就是同步，要控制多个线程对于一个锁的访问获取释放。
获取锁失败的线程都会进入同步队列，而一个lock上可以有多个condition,获取锁成功后还需要等待某个condition的线程进入等待队列，然后释放锁。

同步队列是一个FIFO队列，实现上使用一个双向链表。
等待队列也是一个FIFO队列，实现上使用一个单向链表。

同步队列的基本结构：
{% img /images/syncqueue.png 400 600 SyncQueue %}

加上等待队列后的结构：
{% img /images/waitingqueue.png 400 600 WaitingQueue %}

因为节点可以在同步队列和等待队列之间转化。（同步队列中节点获得锁后，可能发现需要等待condition，进入等待队列尾部；等待队列中节点唤醒后进入同步队列尾部）
JDK中将两个链表的节点的数据结构杂糅在了一起，大致如下：

|属性|描述|
|:-: | -: | 
|Thread thread | 线程引用。
|Node prev | 同步队列使用。前驱同步节点。
|Node next | 同步队列使用。后继同步节点。
|Node nextWaiter| 等待队列使用。后继等待节点。
|int waitStatus| 等待状态。

其中的waitStatus取值包括：
`Cancelled`：1. 同步队列中等待超时或被中断;
`Signal`：-1. 当前节点释放了同步状态或者被取消。
`Condition`: -2. 当前节点等待某个Condition，进入等待队列。
`Propagate`: -3. 下一次共享式同步状态获取将会无条件地被传播下去。(没看懂.TODO)
`Initial`: 0. 初始状态。

同步队列遵循FIFO，AQS中保存了head和tail。
每次唤醒时，唤醒head；（由于每次由已经获取锁的线程完成，只有一个线程，没有并发，因此不需要CAS）
每次新增线程时，用CAS新增更改tail。(各种链表指针操作)


- `自旋`：
同步队列中的线程被唤醒的两种可能：
1. 被中断；
2. 被离开队列的首节点唤醒。
每次被唤醒都会进行一次自旋。
自旋说白了就是一个死循环，首先检查前驱是否是首节点（判断是否是第二种情况），如果是就试图获取锁，获取失败就接着睡等待下次唤醒（等待下次自旋）。
因此并不是只有第二个节点会自旋， 同步队列的所有节点都会自旋，毕竟存在被中断唤醒的可能。

{% img /images/spin.png 400 600 spin %}

**同步状态的获取与释放**
同步状态获取释放相关的方法主要有两类：独占式和共享式。

- 独占式:
1. 模版方法：(AQS中写好的,可以翻看源码研读)
```java
public final void acquire
public final boolean release
private Node addWaiter
private Node enq
final boolean acquiredQueued`
```
2. 需要自己重写的：
```java
protected boolean tryAcquire
protected boolean tryRelease
```


独占式的特点就是：
`tyrAcquire`,`tryRelease`的返回值是`boolean`。
因为同一时刻只能有一个线程占据，因此用`boolean`就能表达了。
相当于资源数只有1，状态只有0,1两种。

- 共享式
{% img /images/shareandmutex.png 400 600 shareandmutex %}

共享式的代码逻辑会复杂一些，如上图所示：
1. 临界区里有共享锁时，只有共享请求能进入；
2. 临界区里有独占锁时，谁都不能进。

共享式的方法：
1. 模版方法：
```java
public final void acquiredShared
private void doAcquiredShared
public final boolean releaseShared
```

2. 自己重写的方法：
```java
protected int tryAcquireShared
protected int tryReleaseShared
```


共享式的特点就是:
`tryAcquireShared`,`tryReleaseShared`返回值是`int`。
因为共享式的资源数可能不为1，状态较多。
`doAcquireShared`方法中判断是否还有资源，用`tryAcquireShared(arg)>=0`来判断，可知自己实现的时候，要注意让`tryAcquireShared`方法返回剩下的资源数。



