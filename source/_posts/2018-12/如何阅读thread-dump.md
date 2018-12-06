---
title: 如何阅读thread dump
date: 2018-12-06 09:24:17
tags:
- java
- jvm 
categories:
- java
- jvm 

---

# 生成thread dump
```
jstack <PID>
```

# 查看thread dump文件
1. 保存成`xxx.tdump`文件,用`jvisualvm`打开;
2. ✔ 用网站打开: http://spotify.github.io/threaddump-analyzer/
3. 用文本编辑器打开.

其中第二个方法，用网站查看最好用。

## jvisualvm查看thread dump
和用文本编辑器的区别只有一个，就是多了语法高亮(字体颜色有区分)。
某个日志如下:
```
"Attach Listener" #16866 daemon prio=9 os_prio=0 tid=0x00007f6574001000 nid=0x2edea waiting on condition [0x0000000000000000]
   java.lang.Thread.State: RUNNABLE
```

各部分含义:

- 线程名称：`Attach Listener`
- 线程类型: `daemon`
- tid: jvm内部线程id;
- nid: 线程id(系统真实)16进制;
- 线程状态: RUNNABLE
- [0x0000000000000000]： 起始栈地址。

## 网站查看thread dump
http://spotify.github.io/threaddump-analyzer
几个改进:
1. 会合并相同栈的线程; (类似一个group by);
2. 会列出锁占用情况(`Synchronizers`面板);
3. Top Methods From 50 Running Threads。
{% img /images/2018-12/groupbythread.png 800 1200 groupbythread %}
{% img /images/2018-12/spotify.png 800 1200 spotify %}
{% img /images/2018-12/topmethod.png 800 1200 topmethod %}

### tip 10进制转16进制
用`top -H -p pid`找到cpu占用高的线程id(10进制)以后，要去thread dump里找具体是哪个nid(16进制),因此需要用这个linux命令:
```
# 把10进制的26转成16进制:
echo 'obase=16;ibase=10;26' | bc
```
或者用printf:
```
printf %x'\n' 26
# 结果是1a
```

### 线程状态

- java中的线程状态:
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

http://xiaoyue26.github.io/2018/02/03/2018-02/java%E5%B9%B6%E5%8F%91%E7%BC%96%E7%A8%8B%E7%9A%84%E8%89%BA%E6%9C%AF-%E7%AC%AC%E5%9B%9B%E7%AB%A0/

- `thread dump`中的线程状态:
示例状态与相应线程名:
```   
// java.lang.Thread.State: RUNNABLE
RUNNABLE: Attach Listener
TIMED_WAITING (parking): HikariPool-1 housekeeper
TIMED_WAITING (on object monitor): Abandoned connection cleanup thread
TIMED_WAITING (sleeping): Hashed wheel timer
WAITING (on object monitor): Finalizer
```
{% img /images/2018-12/thread_dump.png 800 1200 thread_dump %}

状态对应含义详解:
```
Runnable: 就绪或运行中
Wait on condition: sleeping或者等待事件
Waiting for Monitor Entry and in Object.wait(): 在synchronized,等待对象锁
```
其中`synchronized`与`obj.wait()`这对好基友:
对象锁: 每个对象的临界区`Monitor`

### 线程栈示例
```
"Abandoned connection cleanup thread" #81 daemon prio=5 os_prio=0 tid=0x00007f66acbbc000 nid=0x279b in Object.wait() [0x00007f6646a8b000]
   java.lang.Thread.State: TIMED_WAITING (on object monitor)
	at java.lang.Object.wait(Native Method)
	at java.lang.ref.ReferenceQueue.remove(ReferenceQueue.java:143)
	- locked <0x0000000080210720> (a java.lang.ref.ReferenceQueue$Lock)
	at com.mysql.jdbc.AbandonedConnectionCleanupThread.run(AbandonedConnectionCleanupThread.java:43)
```
先lock`0x0000000080210720`,然后wait.
可以想象代码可能是形如:
```
synchronized(obj) {  
       obj.wait();  
}  
```

### Monitor、Entry Set、Waiting Set
`Monitor`的两个set: `Entry Set`, `Waiting Set`.
首先得进入`Entry Set`,才能通往`Waiting Set`，因此要先`Synchronized`才行。
`Entry Set`: `synchronized`
`Waiting Set`: `obj.wait()`// syn后。// 会释放对象锁。
唤醒`Waiting Set`： 在`synchronized`后调用`notify`

### 问题定位
#### Cpu很忙：检查runnable的线程;
1. 隔一段时间，收集几次thread dump;
2. 比较其中runnable的代码变化。

没变化=>可能在死循环;
有变化=>正常。

### Cpu很闲：检查waiting for monitor entry的线程。
1. 隔一段时间，收集几次thread dump;
2. 比较其中waiting for monitor entry的代码变化。

没变化=>可能在死等(死锁); =>可以
有变化=>正常。


### Thin Lock, Fat Lock, Spin Lock:
`Thin Lock`: 某线程通过`CAS`获得`thin lock`,其他线程进入`spin lock`;
`fat lock`:  `monitor_enter`/`monitor_exit`，其他线程进入`wait()`,不消耗cpu;
// 第二个获得锁的人把锁升级成`fat lock`。// 不能回退/降级

`Tasuki`锁：（Oracle的BEA JRockit与IBM的JVM）
1. `thin lock`取消忙等待；
2. `fat lock`可以回退/降级，通过采样数据分析决定。