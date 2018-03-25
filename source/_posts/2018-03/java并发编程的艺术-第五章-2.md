---
title: java并发编程的艺术-第五章(2)
date: 2018-03-25 17:48:16
tags:
- java
- 并发
categories:
- java
- 并发

---




前情提要:
```
(1)中写的主要是Lock接口,Condition接口的api；
然后是自定义Lock时，需要使用的AQS的api以及大致实现、示例、基础数据结构。
```
这部分主要介绍案例：jdk库中提供的重入锁、读写锁以及`LockSupport`。

# java并发编程的艺术-第五章(2)
# 5.3 重入锁
重入锁： 一个线程能否重复获得同一个锁。
该特性需要解决两个问题：
1. 线程再次获取锁。识别获取锁的线程是否为当前占据锁的线程。
2. 锁的最终释放。重复获取n次，则也需要释放n次。

重入锁示例：
1. `synchronized`临界区,同一个线程能够重复进入;
2.  `ReentrantLock`锁，能够重复使用`lock.lock()`.

不可重入锁示例:
前文(1)部分中自定义的锁`Mutex`。


**公平锁与非公平锁**
公平锁： FIFO，竞争锁时需要判断先来后到；
非公平锁： 效率优先，可能有饥饿。同一个线程可能连续获得锁。

- 实现上：
`ReentrantLock`的公平锁的`tryAcquire`方法判断条件比非公平的多了一个`hasQueuedPredecessors`方法，以确保FIFO。

回顾之前同步队列的节点数据结构，是一个双向链表，因此可以判断前驱节点是否存在，即使是因为中断被唤醒节点也可以正确判断自己的位置。 而等待队列是一个单向链表，因此如果节点需要进入到等待队列时，本质上都是非公平的。



# 5.4 读写锁ReentrantReadWriteLock
前文提到的所有`Lock`的实现，依赖于一个状态变量`volatile int state`。本质上都是排他锁。只不过有些实现上通过设定`state`的合法状态范围(`TwinsLock`)，设定了资源的最大数量，让同一个时间能有多个线程同时获取到锁。(需要考虑与可重入特性是否冲突)


读写锁通过对`state`状态变量进行前16位和后16位分割，当作两个状态变量来使用（需要考虑数据类型溢出），从而同时保存了读写状态。

- 读写锁：
1. 同一时刻可以允许多个读线程访问;
2. 写线程访问时：其他读写线程都不能访问；
3. 读线程访问时：读线程可以访问，写线程不能访问。

回顾(1)部分中的独占式和共享式api的区别，可以明白读写锁的实现需要同时实现`tryAcquire`(独占式)和`tryAcquireShared`(共享式)。


读写锁能提供比简单写锁更好的性能。（并发性和吞吐量）

- 读写锁`ReentrantReadWriteLock`的特性：
1. 支持公平或非公平锁；
2. 支持重进入；
3. 支持锁降级。

- 锁降级
锁降级指的是从写锁降级为读锁。
具体流程：
1. 获取写锁;
2. 写数据+do something;
3. 获取读锁;
4. 释放写锁;
5. 读数据+do something;
6. 释放读锁。

那么为什么需要锁降级这个特性呢？
因为需要提高性能。
锁降级的基本思想就尽量减少写锁的持续时间，同时保持这个线程操作的语义不变。

例如：
假如一个线程A需要做的事：
1. 写a=1;
2. 读a,然后计算b=a+1(结果b=2)。

上述过程中其实只有步骤1需要写锁，从步骤2开始只需要读锁就好了。
但如果直接在步骤1后释放写锁，从1到2的时间间隙中，可能被别的线程获取到写锁，然后修改了a的值。这样就改变了线程A操作的原子性。

为了保证线程A操作的原子性，有两种方案：
1. 步骤1和2整个过程都占据写锁；
2. 步骤1结束后，进行锁降级。由于线程A占据读锁后，所有线程无法获取写锁，达到了性能与语义兼顾。

使用锁降级的话，整个过程中所有别的线程都无法获取写锁，但别的线程在后半程能够获取读锁。因此提高了读性能。


## 5.4.1 读写锁的接口与示例
接口： `ReadWriteLock`
jdk实现： `ReentrantReadWriteLock`


`ReadWriteLock`的api:
1. `readLock()`
2. `writeLock()`

 `ReentrantReadWriteLock`的api:
 
 |主要Api|描述|
|:-: | -: | 
|int getReadLockCount()| 读锁被获取的次数(pv). 不去重。
|int getReadHoldCount()| 当前线程获取读锁次数(pv).不去重
|boolean isWriteLocked()| 写锁是否被获取
|int getWriteHoldCount()| 写锁被获取的次数(pv)
 
**案例之Cache**
`ReentrantReadWriteLock`的使用示例，实现一个`cache`:
```
import java.util.HashMap;
import java.util.Map;
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantReadWriteLock;

public class Cache {
    static Map<String, Object> map = new HashMap<>();
    static ReentrantReadWriteLock rwl = new ReentrantReadWriteLock();
    static Lock r = rwl.readLock();
    static Lock w = rwl.writeLock();

    public static final Object get(String key) {
        r.lock();
        try {
            return map.get(key);
        } finally {
            r.unlock();
        }
    }

    public static final Object put(String key, Object value) {
        w.lock();
        try {

            return map.put(key, value);
        } finally {
            w.unlock();
        }
    }

    public static final void clear() {
        w.lock();
        try {
            map.clear();
        } finally {
            w.unlock();
        }
    }
}
```

通过`ReentrantReadWriteLock`生成的`readLock`和`writeLock`，把非线程安全的`HashMap`操作包装成线程安全的，并且尽量保持了并发性能。使用上还是比较简单的，只需要每次加上`finally unlock`即可。

## 5.4.2 读写锁的实现分析

1. 读写状态的设计
由于需要使用AQS来实现读写锁，而AQS成员变量里状态变量只有一个，因此将`state`变量复用为两个变量。`state`本来是一个`int`，把高16位作为读的状态量,低16位作为写的状态量。
由此可以看出读线程最大并发数是`2^16-1`，写线程重入的嵌套深度是`2^16-1`。

读状态: `S>>>16` (无符号右移)
写状态：`S & 0x0000FFFF`

2. 写锁的获取与释放：
写锁获取： S=0(c=0)，没有人获取写锁，也没人获取读锁。
由`exclusiveCount`函数获取写状态。
```
@ReservedStackAccess
protected final boolean tryAcquire(int acquires) {
    Thread current = Thread.currentThread();
    int c = getState();
    int w = exclusiveCount(c);
    if (c != 0) {
        // (Note: if c != 0 and w == 0 then shared count != 0)
        if (w == 0 || current != getExclusiveOwnerThread())
            return false;
        if (w + exclusiveCount(acquires) > MAX_COUNT)
            throw new Error("Maximum lock count exceeded");
        // Reentrant acquire
        setState(c + acquires);
        return true;
    }
    if (writerShouldBlock() ||
        !compareAndSetState(c, c + acquires))
        return false;
    setExclusiveOwnerThread(current);
    return true;
}

@ReservedStackAccess
protected final boolean tryRelease(int releases) {
    if (!isHeldExclusively())
        throw new IllegalMonitorStateException();
    int nextc = getState() - releases;
    boolean free = exclusiveCount(nextc) == 0;
    if (free)
        setExclusiveOwnerThread(null);
    setState(nextc);
    return free;
}
```


3. 读锁的获取与释放
读锁获取： 没有人占据写锁。
```
@ReservedStackAccess
protected final int tryAcquireShared(int unused) {
    Thread current = Thread.currentThread();
    int c = getState();
    if (exclusiveCount(c) != 0 &&
        getExclusiveOwnerThread() != current)
        return -1;
    int r = sharedCount(c);
    if (!readerShouldBlock() &&
        r < MAX_COUNT &&
        compareAndSetState(c, c + SHARED_UNIT)) {
        if (r == 0) {
            firstReader = current;
            firstReaderHoldCount = 1;
        } else if (firstReader == current) {
            firstReaderHoldCount++;
        } else {
            HoldCounter rh = cachedHoldCounter;
            if (rh == null ||
                rh.tid != LockSupport.getThreadId(current))
                cachedHoldCounter = rh = readHolds.get();
            else if (rh.count == 0)
                readHolds.set(rh);
            rh.count++;
        }
        return 1;
    }
    return fullTryAcquireShared(current);
}

@ReservedStackAccess
protected final boolean tryReleaseShared(int unused) {
    Thread current = Thread.currentThread();
    if (firstReader == current) {
        // assert firstReaderHoldCount > 0;
        if (firstReaderHoldCount == 1)
            firstReader = null;
        else
            firstReaderHoldCount--;
    } else {
        HoldCounter rh = cachedHoldCounter;
        if (rh == null ||
            rh.tid != LockSupport.getThreadId(current))
            rh = readHolds.get();
        int count = rh.count;
        if (count <= 1) {
            readHolds.remove();
            if (count <= 0)
                throw unmatchedUnlockException();
        }
        --rh.count;
    }
    for (;;) {
        int c = getState();
        int nextc = c - SHARED_UNIT;
        if (compareAndSetState(c, nextc))
            // Releasing the read lock has no effect on readers,
            // but it may allow waiting writers to proceed if
            // both read and write locks are now free.
            return nextc == 0;
    }
}
```

**4. 案例之锁降级**
锁降级指的是从写锁降级为读锁。
具体流程：
1. 获取写锁;
2. 写数据+do something;
3. 获取读锁;
4. 释放写锁;
5. 读数据+do something;
6. 释放读锁。
```
// 锁降级案例
    public void processData() {
        r.lock();
        if (!update) {
            r.unlock();
            // 1. 获取写锁
            w.lock();
            try {
                if (!update) { // 双检,update状态可能又变化了
                    // do something
                    // 2. 写数据
                    update = true;
                }
                r.lock(); // 3. 获取读锁
            } finally {
                w.unlock(); // 4. 释放写锁（锁降级完成,写锁变成了读锁）
            }
        }
        try {
            // do something
            // 5. 读数据
        } finally {
            r.unlock(); // 6. 读锁最终释放
        }
    }
```
值得注意的是，案例中使用了双检，因此`update`变量应该是`volatile`。


# 5.5 LockSupport工具
回顾前文中的实现层次，自顶向下：
```
1. Lock/Condition接口
2. AQS
3. volatile/CAS/LockSupport
```

其中AQS中除了使用`volatile`变量与`CAS`操作以外，还调用了`LockSupport`以完成等待操作。
例如线程在同步队列中进行自旋等待时，调用的方法：
```
private final boolean parkAndCheckInterrupt() {
        LockSupport.park(this);
        return Thread.interrupted();
}
```
唤醒下一个节点时调用的方法：
```
private void unparkSuccessor(Node node) {
        int ws = node.waitStatus;
        if (ws < 0)
            node.compareAndSetWaitStatus(ws, 0);
        Node s = node.next;
        if (s == null || s.waitStatus > 0) {
            s = null;
            for (Node p = tail; p != node && p != null; p = p.prev)
                if (p.waitStatus <= 0)
                    s = p;
        }
        if (s != null)
            LockSupport.unpark(s.thread);
    }
```

`LockSupport`提供的api:

|主要Api|描述|
|:-: | -: | 
| void park(Object blocker) | 阻塞当前线程. 类似wait。使用时可以park(this)，也可以park其他对象。
| void parkNanos(long t)| 加上超时返回。
| void parkUntil(long deadline)| 最迟deadline时返回。 
| void unpark(Thread thread) | 唤醒特定线程。
| Object getBlocker(Thread t) | 获取某线程调试对象，如果未阻塞则为null。

上述api也可以不带blocker参数。blocker参数仅仅是用于调试和系统监控。
 
`LockSupport`提供的`park`/`unpark`类似于`wait`/`notify`，都是等待/通知的模式。主要存在以下几点不同：
1. `park`还可能在没有被唤醒的时候返回,因此必须在循环中重新检查返回条件。这种设计是一种忙碌等待的优化，效率介于快速自旋与`wait`之间,灵敏度介于快速自旋与`wait`之间。
示例用法（检查返回条件）:
```
public final void await() throws InterruptedException {
            if (Thread.interrupted())
                throw new InterruptedException();
            Node node = addConditionWaiter();
            int savedState = fullyRelease(node);
            int interruptMode = 0;
            while (!isOnSyncQueue(node)) {
                LockSupport.park(this);
                if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
                    break;
            }
            if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
                interruptMode = REINTERRUPT;
            if (node.nextWaiter != null) // clean up if cancelled
                unlinkCancelledWaiters();
            if (interruptMode != 0)
                reportInterruptAfterWait(interruptMode);
        }
```

2. `unpark`可以先于`park`调用，而`notify`不能先于`wait`。`unpark`相当于赋予线程一个许可，最多缓存一个，等待下一次`park`时可以直接通过。
3. `unpark`可以精确唤醒某个线程,而`notify`只能随机唤醒一个，或者唤醒全部。


## LockSupport实现浅析
总结：
```
park对象用于实现；
blocker对象用于调试。
```

`park`与`unpark`的实现都是委托给了一个`Unsafe`对象`U`实现的：
```
// Hotspot implementation via intrinsics API
private static final Unsafe U = Unsafe.getUnsafe();
public static void park() {
        U.park(false, 0L);
    }
```

`park`方法面向的主体是`Thread`，每个线程内有一个`Parker`对象以承载相应的阻塞操作；
`wait`方法则依赖的是每个对象的内置锁实现。
因此两者是正交的。

可以查阅`hotpot`的源代码进一步深入其`Parker`的实现。

**Blocker参数**
带`Blocker`参数的`park`方法:
```
public static void park(Object blocker) {
        Thread t = Thread.currentThread();
        setBlocker(t, blocker);
        U.park(false, 0L);
        setBlocker(t, null);
    }
private static void setBlocker(Thread t, Object arg) {
        // Even though volatile, hotspot doesn't need a write barrier here.
        U.putObject(t, PARKBLOCKER, arg);
}
```
可以看出区别是多了`setBlocker`函数的调用，而且`Blocker`参数在阻塞结束后会被清空。
此外，`Blocker`与`Parker`类似，都是每个线程有一个。设定的逻辑是在该线程`t`的`PARKBLOCKER`偏移量中填入对象`blocker`的引用。

使用`blocker`的话，在线程阻塞时进行线程dump，可以获得`blocker`的信息，方便调试和监控。
类比，如果线程因为`synchronized(this)`而阻塞，线程dump的时候是可以获得`this`的信息的。


**实验`Blocker`：**

```
public class ParkWithBlockerTest {
    public static class UnparkerAndReciever extends Thread {
        Thread main;

        UnparkerAndReciever(Thread main) {
            this.main = main;
        }
        private void sleep(int secends){
            try {
                TimeUnit.SECONDS.sleep(secends);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
        @Override
        public void run() {
            sleep(1);
            LockSupport.unpark(main);
            sleep(1); // 注释此行则出现结果1

            Object blocker = LockSupport.getBlocker(main);
            if (blocker != null && blocker instanceof String) {
                System.out.println(blocker); // 结果1
            }
            else{
                System.out.println("blocker=null or not a String"); // 结果2
            }


        }
    }

    public static void main(String[] args) {
        Thread mainThread = Thread.currentThread();

        UnparkerAndReciever unparkerAndReciever = new UnparkerAndReciever(mainThread);
        unparkerAndReciever.start();
        LockSupport.park("string in park");
    }
}
```
