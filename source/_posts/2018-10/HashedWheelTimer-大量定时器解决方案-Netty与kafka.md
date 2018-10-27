---
title: HashedWheelTimer-大量定时器解决方案(Netty与kafka)
date: 2018-10-27 20:16:33
tags: 
- java
- netty
- kafka
- 定时任务
categories:
- java
- netty

---

# 需求:
有大量定时任务需要执行，精度要求不高，可以允许延迟执行。
可能的场景: 每个连接的超时事件、每个请求的超时事件。

# 方案1: 
每个定时任务设置一个定时器、或一个`Scheduled`,`DelayedQueue`和实现`Delayed`接口的线程。
缺点: 开销太大;
优点: 少量任务时精度较高。

# 方案2: 超时集
将相同时间间隔的任务组织成一个集合。每个集合分配一个计时器（thread）。
缺点: 相同时间间隔难以满足，性能不够。

# 方案3: HashedWheelTimer： 时间轮算法(Netty4工具类)
设计一个虚拟的哈希表组织定时任务。
优点: 默认只用一个thread,开销小;
缺点:
1. 精度降低到`tickDuration`粒度;
2. 定时任务不能太耗时;(解决方案: 可以在定时任务里异步处理)。

## 基本架构
{% img /images/2018-10/timewheel.jpg 400 600 timewheel %}
如上图所示即为时间轮。
引入术语:
```
tick: 时间轮里每一格;
tickDuration: 每一格的时长;
ticksPerWheel: 时间轮总共有多少格.
newTimeout: 定时任务分配到时间轮
```

### 定时任务分配到时间轮
分配流程:
1. 计算定时任务deadline = `System.nanoTime() + unit.toNanos(delay) - startTime`;
2. 计算定时任务放入第几格: 
```java
long calculated = deadline / tickDuration; // 需要计算几次
final long ticks(还需要走几格) = Math.max(calculated, 当前tick次数); // 一般就等于calculated，这里只是为了确保不在以前走过的格子里（这样的话这个任务永远不会执行而且移除不掉了）
stopIndex = (int) (ticks & mask); // 相当于% wheel.length，因为长度是2的幂。
```
3. 计算定时任务第几轮被调用: `remainingRounds = (calculated - tick) / wheel.length;`
4. 放入时间轮stopIndex位置中任务链表: (remainingRounds,task)：
```java
HashedWheelBucket bucket = wheel[stopIndex];
bucket.addTimeout(timeout); // 加入链表尾部。
```

当这个时间轮开始运行的时候（也就是开始计时，执行定时任务了），每次跳转一个tick,都会检查这个tick里的定时任务,如果定时任务轮次应当执行，则执行对应任务。


使用的示例代码: 
```java
    /**
     * tickDuration: 每 tick 一次的时间间隔, 每 tick 一次就会到达下一个槽位
     * ticksPerWheel: 轮中的 slot 数
     */
    @Test
    public void testHashedWheelTimer() throws InterruptedException {
    // 1000ms一格,一轮16格（一般是2的N次幂，这样可以把hash转换为&0xFFFF）
        HashedWheelTimer hashedWheelTimer = new HashedWheelTimer(1000,TimeUnit.MILLISECONDS, 16 );
        System.out.println(LocalTime.now()+" submitted");
        Timeout timeout = hashedWheelTimer.newTimeout((t) -> {
            new Thread(){
                @Override
                public void run() {
                    System.out.println(new Date() + " executed");
                    System.out.println(hashedWheelTimer);
                    try {
                        TimeUnit.SECONDS.sleep(100);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                    System.out.println(new Date() + " FINISH");
                }
            }.start();
        }, 5, TimeUnit.SECONDS);

        hashedWheelTimer.newTimeout((t) -> {
            new Thread(){
                @Override
                public void run() {
                    System.out.println(new Date() + " TASK2 executed");
                    System.out.println(hashedWheelTimer);
                    try {
                        TimeUnit.SECONDS.sleep(10);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                    System.out.println(new Date() + " TASK2 FINISH");
                }
            }.start();
        }, 15, TimeUnit.SECONDS);

        TimeUnit.SECONDS.sleep(500);
    }
```


`HashedWheelTimer`源码:
https://github.com/netty/netty/blob/4.1/common/src/main/java/io/netty/util/HashedWheelTimer.java

理论基础:(操作系统内核的定时器)
http://www.cs.columbia.edu/~nahum/w6998/papers/sosp87-timing-wheels.pdf


# 方案4： 层级时间轮(Kafka使用)
上述简单时间轮的算法存在的**缺点**:
对于延迟很长时间的任务,该方案存在较多的cpu空转。一种可能的方案是增大`tickduration`,但是很难兼顾精度和性能。

一种解决方案: 层级时间轮。
思路是模拟现实中的钟表,现实中的钟表有时针、分针、秒针,相当于不同`tickDuration`的时间轮。
架构图如下:
{% img /images/2018-10/timewheel2.jpg 400 600 timewheel2 %}

不同于`Netty4`中每个任务存放自己应到执行的轮次,层级时间轮把高轮次的定时任务分配到高层的时间轮中:层数越高=>`tickDuration`越大。

假如定义最低层的时间轮的层号为0,高一层的为1,依次类推,只有n-1层的定时任务到期后,才会取出n层时间轮的定时任务处理（要么直接执行,要么降级,要么放回时间轮）。

(如果让层级时间轮每层的`tickDuratino`相同，则退化为方案3的普通时间轮。)

同时为了辅助推进时间轮的指针,使用`DelayQueue`存放最近到期的定时任务

## 双向链表
时间中每一个格中使用的数据结构为双向链表。
好处是只要持有其中某个节点，就可以在`O(1)`时间内进行插入或者删除操作。

## 到期操作
由`DelayQueue`辅助推进指针: 
1. `DelayQueue`中优先级队列的队首最近到期的定时任务。处理结束后,由`leader`线程等待下一个定时任务的时间。因此推进步长由定时任务的间隔决定,没有cpu空转的现象。
2. 推进所有时间轮的指针,对于遇到的定时任务,要么插入低层时间轮,要么删除(执行)。



## 插入操作(删除类似)
插入新的定时任务:
1. 插入时间轮: 根据到期时间,可以算出具体应该放在哪一层时间轮的哪一格,得到对应的双向链表引用。如果这一格还是空的,需要新建一个,然后插入到`DelayQueue`中(`O(logN)`时间复杂度)。(参见`DelayQueue`原理,如果时间低于`DelayQueue`的最小延时任务,会提前唤醒里面的`leader`线程)
2. 插入`DelayQueue`: 根据上一步中得到的双向链表引用,往里面插入新的定时任务（`O(1)`）。

## 总结
层级时间轮通过不同`tickDuration`的时间轮,以较小的空间映射到一个很大的时间范围,兼顾了精度和性能。
**插入删除的时间复杂度:** 
时间复杂度其实取决于有效格子的数量,因为`DelayQueue`存放的是有任务的格子,也就是双向链表的数量。低层用的是优先级队列(最小堆),假如有效格子的数量是`m`，则复杂度为`O(log(m))`。
实际中m一般远小于n，因此性能有很大的提高。
**理想情况**:定时任务的时间间隔分布能尽量让它们位于相同格子中。 (`O(1)`)
**最坏情况**:所有双向链表都只有一个任务(均匀地分布在不同格子中)。(`O(logN)`)

实际业务中,定时任务一般都服从对数正态分布,因此每次插入删除时间复杂度是接近`O(1)`的。
