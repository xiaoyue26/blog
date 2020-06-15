---
title: jdk11下g1收集器使用
date: 2020-06-15 09:37:21
tags: 
- java
- jvm
categories:
- java
- jvm
 
---

# WHAT: g1是什么?
g1是一个jdk9推荐默认使用的垃圾收集器。


# WHY: 为什么要使用g1
主要优点：收集高吞吐、没有内存碎片、收集时间可控。

G1出来之前，OLTP应用之前一般使用CMS收集器，达到暂停时间短的效果。
CMS: https://www.jianshu.com/p/fed80fdba376

CMS收集器的缺点:
1.有内存碎片: 标记清理算法容易留下碎片，可以用参数在几次full gc以后进行一次压缩；`-XX:CMSFullGCsBeforeCompaction=0`: 每次都压缩;
2.full gc风险(foreground): 业务线程请求分配内存，但是内存不够了，于是可能触发一次CMS GC，这个过程就必须要等待内存分配成功后业务线程才能继续往下面走，因此整个过程必须STW，所以这种CMS GC整个过程都是STW，相当于full gc了;

cms触发回收: `-XX:CMSInitiatingOccupancyFraction=75 -XX:+UseCMSInitiatingOccupancyOnly`: old区占75%的时候满足回收条件，检查这个条件的线程:`-XX:CMSWaitDuration=2000`(默认2秒)。

g1以前的大部分收集器包括cms，都需要程序员手动设置新生代和老生代，g1则会自动调节eden region和old region的占比，以达到设定的暂停时长目标，因此更加智能。
g1的目标是取代CMS，因此我们有必要了解一下g1。

# HOW: G1怎么工作的、原理

{% img /images/2020-06/g1_region.png 800 1200 g1_region %}

g1以前的收集器是新生代+老生代的布局，g1则是先分<=2048个region，然后这些region可以用于eden region\old region\humongous region\suvivor region。
相当于空间布局更加细致了。

eden: 和以前的新生代eden类似；
old: 和以前的老生代类似；
humongous：和以前的大对象直接进old区类似；
suvivor: 和以前的双缓冲区类似。


g1可以通过统计信息，动态调节eden region和old region的比例，达到设定的暂停时长目标。

## G1中的关键过程
G1垃圾收集器工作过程中有几个关键的过程：
1.对象分配: 有内存使用才有内存的回收；
2.YGC;
3.全局并发标记周期;
4.Mixed GC;
5.FGC: 具体实现不是G1负责，只是作为兜底使用。

### 对象分配
大部分到eden，少部分直接到 Humongous region， 统计时算作old区占用。
Humongous: >=0.5*region size的对象会直接分配一个独立的Humongous region。
相关参数: `-XX:G1HeapRegionSize=16M`

### YGC
STW, 并发复制，一部分到old（年龄）；  Ygc末尾，重新计算edan区,survive区大小
#### 触发时机
eden region用完。

### 全局并发标记周期
即concurrent marking cycle: 主要为old region/Humongous region回收服务。

#### 1.初始标记 
STW, initial mark,找根；

这个过程需要跟混YGC，在YGC的末尾触发，因为需要STW，而YGC也需要STW，为了减少STW的次数，就让初始标记阶段直接跟混YGC的STW。

理由也很简单，因为正常的服务YGC肯定比较频繁，比OLD区满的频率大多了。而且大部分增大old区的对象都在YGC之后从young区移动到old region，除了直接分配的大对象之外。而大对象毕竟是小概率、低频事件。

#### 2.并发扫描
并发扫描从新生代出发的直接可达性，完成后才能下一次YGC,不然白扫了

#### 3.并发标记
扫描间接可达性；

#### 4.最终标记
STW, 避免浮动垃圾；

#### 5.并发清理
STW, 回收完全空闲的region。(比如直接回收Humongous region)
这个过程不会产生碎片，因为是整个region回收的。
并不是完全空闲的region的处理: 交给Mixed GC。

### Mixed GC
#### 触发时机
标记阶段结束后，可以知道old region有多少可以被回收；YGC之后,看浪费占比(可回收未回收),太浪费的话就MixedGC;

相关参数:
1. `G1HeapWastePercent=10`: YGC之后,看浪费占比(可回收未回收),太浪费的话就MixedGC;
2. `G1MixedGCLiveThresholdPercent=85`: 超过85%占用率的region就不加入CSet了，性价比太低;
3. `G1MixedGCCountTarget=8`: 连续可以进行多少次MixedGC;
4. `G1OldCSetRegionThresholdPercent`: 一次Mixed最多选多少region进CSet;

### FGC
G1不直接提供full gc，< jdk10是调的serial old，我们用的jdk11是调并发的收集，都需要STW。

#### 触发时机
由于标记阶段不能进行Mixed GC，所以如果标记阶段堆被塞满了，就会触发FGC。(一般是非STW阶段,也就是第二阶段并发扫描和第三阶段并发标记，因为STW阶段不会分配新的对象)

### Evacuation
非完全空闲的region的处理，都是压缩复制到另一个region，G1称这个为Evacuation。（翻译过来就是对象的疏散）

## 小结
从上述过程可以总结占用、增多各种region的事件；回收、减少各种region的事件。
eden region增多: 普通小对象的分配;
eden region的减少: YGC;
Humongous region的增多: 大对象的分配，直接进H region;
Humongous region的减少: concurrent marking cycle的最后阶段（并发清理），回收完全空闲的region; 
old region的增多: YGC末尾，随着对象年龄晋级到old region;
old region的减少: concurrent marking cycle的最后阶段（并发清理），回收完全空闲的region; 以及Mixed GC: 回收部分占用的region。

# 实战
## 环境配置
容器CPU: 4
容器内存: 8G
JDK: 11 
JVM参数: 未设置

## 现象
通过命令:
```shell
nohup jstat -gcutil <pid> \
> gcutil.log 2>&1 & echo $! > gcutil.pid
```
发现old区增长比young区快。

执行命令:
```shell
jmap -histo:live <pid> | head
```
触发FGC、STW，内存占用突然掉下来。

观察内存自然回收前一刻的内存占用，可以看到old区占用大概为43.49%。而默认的参数`-XX:InitiatingHeapOccupancyPercent=45`。
G1这里的`InitiatingHeapOccupancyPercent`指的是old区占整个堆的比例，和CMS的不同。也有资料认为这个是指整个堆的占用/总大小，但其实统计的时候都统计的是YGC以后的堆的占用/总大小，
也就等于old区的使用/整个堆的大小。

所以这种情况下young区增长比old区快的原因是大对象的分配。
大对象region的清理依赖标记阶段最后的清理，而标记阶段的初始mark依赖YGC的末尾。

最后因为没有几乎YGC，所以大对象很容易就一直堆积着、而且由于young区太大了，old区的占用很高也触发不了标记阈值。

## 调优参数
如果要维持old区在gcutil视角下不超过90%，可以通过简单的计算，将InitiatingHeapOccupancyPercent调整为37%。（仅当前case）

```shell
-Xmx4096M -Xms4096M -XX:G1HeapRegionSize=8M -XX:MetaspaceSize=512M -XX:MaxMetaspaceSize=512M -Xlog:gc*:gc.log -XX:InitiatingHeapOccupancyPercent=37
```

{% img /images/2020-06/gcutil2.png 800 1200 gcutil2 %}

调整以后old区维持98%, YGC次数增多。
但这个参数仅第一次有用，因为之后G1会根据暂停时间目标来调整不同region的比例，因此并不能长期解决。

### 调参2
我们这个case因为是空转服务，而且Eden区基本不使用，光怼Old区了（大对象）。可以通过调节大对象阈值，让Eden区增长速度稍微快于Old区(IHOP为45%)。

相同参数: `-Xmx4096M -Xms4096M -XX:MetaspaceSize=512M -XX:MaxMetaspaceSize=512M -Xlog:gc*:gc.log` 

各种姿势的结果：

| 思路       | gc耗时(ms) | YGC | Mixed GC | FGC | 差异参数                                                                              |
|------------|------------|-----|----------|-----|---------------------------------------------------------------------------------------|
| 拒绝大对象 | 2260       | 31  | 0        | 0   | -XX:G1HeapRegionSize=16M -XX:InitiatingHeapOccupancyPercent=37                        |
| 拒绝大对象 | 5390       | 52  | 0        | 0   | -XX:G1HeapRegionSize=32M -XX:InitiatingHeapOccupancyPercent=37                        |
| 停顿小     | 5688       | 128 | 0        | 0   | -XX:G1HeapRegionSize=8M -XX:InitiatingHeapOccupancyPercent=37 -XX:MaxGCPauseMillis=50 |
| 不让调IHOP | 3507       | 128 | 1        | 0   | -XX:G1HeapRegionSize=8M -XX:InitiatingHeapOccupancyPercent=37 -XX:-G1UseAdaptiveIHOP  |


下面解释4个试验结果：

结果1： 16M的region，所以大于8M的对象会分配一个Humongous region，否则在edan区分配，由于Humongous region也在统计中算到了old region里头，所以我们这样操作以后会减少统计结果中old区的占用率。16M配置下，看gc.log已经完全没有了Humongous region，因此这样配置就肯定不会触发报警了。

结果2：32M的region，类似于16M的，0个Humongous region，但由于region比较大，碎片多，比较浪费, YGC比较多，gc耗时比较大；

结果3：由于G1会动态调整young区\old区的大小比例，young在5%~60%之间调整，如果young在60%，那么即使old区填满了也才40%，无法达到IHOP，也就无法通过IHOP触发标记阶段，只能通过YGC触发；因此这里我们把自动调整的关掉，这样young区就不会太大，就能触发IHOP的阈值了。这种情况下由于region大小没有调大，YGC次数没有太大变化；而由于没有使用自动调整IHOP，old区很满以后会触发标记阶段，然后G1发现回收young以后，浪费的空间仍然大于G1HeapWastePercent参数，于是就进行Mixed GC，回收old区。所以这种配置下有1次MixedGC。

结果4：这个思路也比较类似思路3，由于G1是根据停顿时间来调整young区/old区大小的，我们把停顿时间设定得超小，它就会把young区变小已达到停顿时间的要求，同时也会更频繁得YGC，Humongous可以在标记阶段的末尾得到清理，所以这种配置下也不会触发Old区的长期占用高的报警。同时由于停顿时间超小，YGC次数变得很高，总耗时也很高。



以上四种配置，真正在线上只有配置1是实用的。因此如果理解了G1的过程，其实调参可以只调很少。


## 其他: Metaspace
除了上述区域，还有一个元数据区域。通过`jstat -gcutil`可以看到它的占用率一般是在98.1%左右，比较高。但实际上元数据区的占用率是越高越高的，因为它的实际含义类似于(1-碎片率)，占用率98.1%，差不多相当于碎片率是1.9%。因为程序工作一段时间后，metaspace基本就不怎么增长了，所以基本不用我们操心，最多只需要将初始metaspace的大小设大一点，避免它增长到稳定之前触发full gc。可以先启动一次，看看稳定的时候metaspace的大小，然后向上取2的幂，设置大小。不要以为把这个值设大2倍，占用率会下降一半，metaspace是用多少申请多少的，所以即使设大了也不会影响占用率的。
```shell
-XX:MetaspaceSize=512M -XX:MaxMetaspaceSize=512M
```
元数据相关参见: http://lovestblog.cn/blog/2016/10/29/metaspace/


# 参考资料
https://www.oracle.com/cn/technical-resources/articles/java/g1gc.html
https://www.jianshu.com/p/0f1f5adffdc1
https://tech.meituan.com/2016/09/23/g1.html
