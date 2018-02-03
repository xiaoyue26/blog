---
title: java并发编程的艺术-第四章
date: 2018-02-03 21:55:23
tags: java
categories:
- java
- 并发
---


# 第四章 JAVA并发编程基础
## 4.1 线程简介
线程(轻量级进程): 现代操作系统调度的最小单位.
线程共享的存储: 堆
线程独占的存储: 栈(局部变量,方法参数),PC,堆的`ThreadLocal`区

JAVA程序天生多线程: 执行`main`方法的是一个名字为`main`的线程.
天生的线程:
1. `Signal Dispatcher`: 分发处理发送给JVM信号的线程;
2. `Finalizer`: 调用对象`finallize`方法的线程;
3. `Reference Handler`: 清除`Reference`的线程;
4. `main`: `main`线程,用户程序入口.

要查看上述线程,可以用JMX打印出来:
```
ThreadMXBean threadMXBean = ManagementFactory.getThreadMXBean();
ThreadInfo[] threadInfos = threadMXBean.dumpAllThreads(false, false);
for (ThreadInfo ti : threadInfos) {
    System.out.println(ti.getThreadId()+":"+ti.getThreadName());
}
/*
5:Monitor Ctrl-Break
4:Signal Dispatcher
3:Finalizer
2:Reference Handler
1:main
*/

```

如果懒得写上述代码,也可以使用IDE的debug功能,例如在`Intellij idea`中打个断点,就直接可以在`debug`标签页看到1-4号线程的栈帧了.(5号看不见,原因未知.TODO)
{% img /images/ide-thread.png 400 600 ide-thread %}

### 4.1.3 线程优先级(很可能被操作系统忽略)
线程优先级: 整型变量`priority`
优先级范围: 1~10
默认优先级: 5
设定策略:
1. cpu密集线程=>设定较低优先级;(需要cpu时间长,防止它独占太久)
2. io密集线程=>设定较高优先级(相当于cpu占用时间不长的线程,反而可以优先给它,一种短作业优先的逻辑).
// 短作业优先可能导致长作业饿死,因此上述策略下,如果IO密集线程特别多,就不好了.


### 4.1.4 线程的状态
- `Runnable`
Java把操作系统中的`就绪`和`运行中`统称为`Runnable`.

线程状态转化大致如下:
1. `New`=>`Runnable`=>`Terminated` // 理想状态
2. `Runnable` => `Blocked`/`Waiting`/`Time_Waited`=>`Runnable` // 可能误入的歧途

状态|说明
--|--|
New|创建线程后,调用`start()`前.
Runnable|就绪和运行中.
Terminated|终止. 执行完毕
Blocked|阻塞.  阻塞于锁. (`synchronized`)
Waiting|等待.  等待其他线程的中断或者通知.(`Lock`类)
Time_Waiting| 超时等待. 比Waiting多一个超时返回功能.(`Lock`类)

**查看某个java程序目前各线程状态:**
1. 先用jps查看该进程的进程id;
2. 运行命令jstack <id>即可.

或者用`kill -3 <id>`命令让进程把`threadDump`信息输出到标准输出.(可以之前让进程把标准输出重定向到日志文件中)
或者用IDE的`threadDump`按钮也可以. 

### Runnable与其他状态的转化
**1.Waiting**
`Runnable`=>`Waiting`: // 主动等待某个对象
```
obj.wait()
obj.join()
LockSupport.park()
```

`Waiting`=>`Runnable`: // 被别人中断或通知
```
obj.notify() // 必须在wait之后调用才有效
obj.notifyAll()
LockSupport.unpark(Thread) // 在park之前调用也有效.(会累计1个,但不会累计2个)
```

**2.Time_Waiting**
`Runnable`=>`Time_Waiting`: // 基本就是比Waiting多个时长
```
obj.wait(long)
Thread.join(long)
LockSupport.parkNanos(long)
LockSupport.parkUntil(long)
Thread.sleep(long)
```

`Time_Waiting`=>`Runnable`: // 与Waiting完全一样
```
obj.notify()
obj.notifyAll()
LockSupport.unpark(Thread)
```

**3.Blocked**
`Runnable`=>`Blocked`:
```
synchronized(xx)// 没获取到锁
```
`Runnable`=>`Blocked`:
```
synchronized(xx)// 获取到了锁
```

### 4.1.4 Daemon线程
守护线程,用作后台调度以及支持性工作.
换句话说,是为普通线程服务的,如果普通线程不存在了(运行结束了),`Daemon`线程也就没有存在的意义了,因此会被立即终止.
> 立即终止发生得非常突然,以至于Daemon线程的`finally`方法都可能来不及执行.


设定线程为`Daemon`的方法:
```
thread.setDaemon(true);
```


## 4.2 启动和终止线程
### 4.2.1 构造线程
线程的构造内容包括:
1. 父线程; (创建它的线程) // 下面的属性默认值均与父线程一致: 
2. 线程组;
3. 是否守护线程;
4. 名字;
5. ThreadLocal内容. (复制一份父线程的可继承部分)

构造完成后,在堆内存中等待运行.

### 4.2.2 启动线程
- start方法的含义:
当前线程(父线程)同步通知JVM虚拟机,在线程规划期空闲时,启动线程.


### 4.2.3 中断
每个线程的中断标识位:
- `true`: 被中断. (收到了中断信号) 
- `false`(初始值): 没中断,或已经运行结束.

容易混淆的几个方法:
```
obj.interrupt();// 中断某线程.把它的中断标志改为`true`.
obj.isInterrupted(); // 查询是否中断
Thread.interrupted();// 把当前线程的中断标志重置为`false`. 
```

除了`Thread.interrupted()`,还有一些方法抛出`interruptedException`前也会清除中断标志(置为`false`),以表示自己已经处理了这个中断.(如`sleep`方法.)

### 4.2.4 废弃方法: suspend(),resume(),stop()
- suspend(): 挂起(暂停), 不释放锁.
- resume(): 恢复(继续)
- stop(): 停止,太突然,可能没释放资源.

### 4.2.5 安全的终止/暂停的方法
使用中断.
例如:
```
public class TestCancel2 {
    class PrimeProducer extends Thread {
        private final BlockingQueue<BigInteger> queue;

        PrimeProducer(BlockingQueue<BigInteger> queue) {
            this.queue = queue;
        }

        public void run() {
            try {
                BigInteger p = BigInteger.ONE;
                while (!Thread.currentThread().isInterrupted()) {// 检查是否中断.
                    queue.put(p = p.nextProbablePrime());
                }
            } catch (InterruptedException e) {
                e.printStackTrace();
                /*
                 * 一般处理策略:
                 * 1. 捕获;
                 * 2. do 自定义存盘工作;
                 * 3. 接着往外抛,提醒调用者.
                 * ( Thread.currentThread().interrupt();
                 * )
                 *
                 * 本代码中由于不需要提醒调用者,因此没有接着往外抛.
                * */
            }

        }

        public void cancel() {
            interrupt();
        }
    }

}

```
这部分在<并发编程实战>第七章有详细讨论.
根据具体情况的不同,有多种解决方案.
1. 简单情况: 直接轮询标志位;
2. while中操作可能阻塞: (1)检查while中每一行;(2)while中只提交任务,起另外的线程执行任务;
3. 能中断但不能取消的任务: 保持中断状态,直到收到继续信号;
4. 其他....
详见:
https://github.com/xiaoyue26/scala-gradle-demo/tree/master/src/main/java/practice/chapter7
