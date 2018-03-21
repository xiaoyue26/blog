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

|Api|备注|
|:-: | -: | 
|lock()| 阻塞式地获取锁。只有在获取到锁后才处理interrupt信息。
|lockInterruptibly()| 获取锁。(可中断)
|tryLock()| 非阻塞地获取锁。不论成败立即返回。
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

# 5.2 队列同步器
队列同步器：AQS(AbstractQueuedSynchronizer)
基于模版方法模式，队列同步器中自顶向下的3类方法：
1. 模版方法；（固定，调用2，3的方法）
2. 可重写的方法（调用3的方法）；
3. 基础方法。

-- to be continue
