---
title: java并发编程实战笔记-4-5章
date: 2017-02-25 19:44:30
tags: java
categories:
- java
- 并发
---

# 对象的组合

# 设计线程安全类
三个步骤:
> 1. 找出构成对象状态的所有变量;
2. 找出约束状态变量的不变性条件;//收集同步需求,例如哪些操作必须是原子性的.
3. 建立对象状态的并发访问管理策略.

# 包装类
常见容器ArrayList不是线程安全的, 但可以通过Collections.synchronizedList方法转化成一个线程安全的容器.(实现上使用装饰器模式,封装对于底层对象的访问).
此时,只要这个包装类持有数据对象的唯一引用,则可以保证容器的安全.

# 监视器模式
通过一个私有的锁保护状态:
```java
public class PrivateLock {
    private final Object myLock=new Object();
    @GuardedBy("myLock")
    JRSUIConstants.Widget widget;
    void someMothode(){
        synchronized (myLock){
            // do some thing
        }
    }
}
```

# 对象的组合
1. 一个没有成员的对象A,无状态,因此是线程安全的;
2. 当A中增加一个成员,如:
```java
public class A{
private final AtomicLong aa=new AtomicLong(0);
}
```
因为引用不可变,且AtomicLong是线程安全的,因此A依然是安全的.
但如果增加多个线程安全成员,当且仅当它们独立的时候是安全的.
如果不独立,例如某些aa和bb的取值组合是不合法的,则是不安全的.
(例如上界下界都是AtomicLong.但不独立.)


## 委托:
> 此时A是否线程安全取决于aa,换言之,A的安全性委托aa来保证. 

综上可知,尽量委托给一个线程安全的类或容器,避免所托非人.XD

# 基础构建模块
将线程安全委托给一个JDK实现好的同步容器后,如果需要进行复合操作,而且需要这几个操作具有原子性,依然需要自己加锁.这个时候就得知道底层的同步容器原先使用的锁是什么,才好照着写.

//每当遇到难题,都可以考虑生成一个静态镜像,线程局部变量,避开这个难题.实在不行只好升级手段,增加复杂度.

## 迭代器与ConcurrentModificationException
> 同步容器的迭代器在迭代之前会获取容器的锁,
如果迭代过程中,检测到容器有修改,则会抛这个异常.
(实现上,是通过计数器实现的,而且没有同步(性能考虑),因此也有可能没有意识到已经修改了,读了失效数据.)

- 隐藏迭代器
容器类的toString方法.直接打印容器的时候,会触发对容器内所有内容的迭代.
其他类似的方法有:
hashCode,equals,ContainsAll等等.
以及容器整体作为一个key,存入另一个容器时.


## 同步容器和并发容器

### 同步容器:
> HashTable,Vector,Collections.synchronizedXxx包装的.
- 强一致性.
- 迭代时锁容器,不允许修改.

### 并发容器:
> ConcurrentMap,CopyOnWriteList,BlockingQueue,ConcurrentLinkedQueue.
- 弱一致性.(size,isEmpty只返回近似值)
- 迭代时一般只锁局部,容忍修改.
- 性能高.

基本上应该使用并发容器代替同步容器. 

## 两种队列的使用场景
1. `BlockingQueue`:
生产者消费者,queue.put(xxx),queue.take();
2. `BlockingDeque`:(双端队列)
工作队列\工作觅取(fork-join);完成了自己的工作队列后,从别人队列的尾巴觅取新的工作,均摊工作量.

## 阻塞方法与中断方法

阻塞状态:
> BLOCKED,WAITING,TIMED_WAITING

如果一个方法签名抛出`InterruptedException`,则说明它是阻塞方法.
因为一个阻塞方法一般会被中断打断.

中断: Interrupt
查询线程是否中断: interrupt方法;
中断线程:   也是  interrupt方法. 

当你的方法catch到了一个`InterruptedException`,两种处理方法:
1. 传递. 接着往外抛;
2. 恢复中断. 
当已经是最外层的时候,(在Runnable这一层了)
就不能往外抛了,再抛线程就挂了.
这个时候保持中断状态就好:(恢复中断)
```java
catch(InterruptedException e ){

Thread.currentThread().interrupt();
}
```

# 同步工具类
可以用于同步线程的工具类,包括:
>BlockingQueue 阻塞队列
Semephore 信号量
Barrier 栅栏
Latch 闭锁

## 闭锁 Latch
作用相当于一道门,所有线程得等这扇门打开才能继续运行.
Latch是一次性的,打开后就不能再关上. 

> - 使用的时候就像主线程设定了好了一扇门挡住起跑线,把所有工作线程挨个释放,一头撞到门上卡住了;等到某个时机放下门,瞬间释放所有线程;
- 设置一扇门挡住主线程,让主线程等待子线程跑完;
- 结束的时候,每一个线程到达终点就把计数器减一,直到大家都结束,再在终点释放主线程.

```java
final CountDownLatch startGate=new CountDownLatch(1);
// 开始的门,只需主线程一人下令countDown,因此计数器为1;
final CountDownLatch endGate=new CountDownLatch(nThreads);
// 结束的门,需要n个子线程countDown,因此计数器为n. 
```


- 完整代码
```java
public class TestLatch {

    public long timeTasks(int nThreads, final Runnable task) throws InterruptedException {
        final CountDownLatch startGate = new CountDownLatch(1);// countDown一次就能打开
        final CountDownLatch endGate = new CountDownLatch(nThreads);//countDown nThreads次才能打开(所有线程都countDown过)

        for (int i = 0; i < nThreads; i++) {// 先全部放出去,然后一头撞在startGate上;
            Thread t = new Thread(() -> {
                try {
                    startGate.await();// 一头撞在startGate上
                    try {
                        task.run();
                    } finally {
                        endGate.countDown();//每个线程countDown一次
                    }
                } catch (InterruptedException e) {
                    e.printStackTrace();//ignore
                }
            });
            t.start();
        }
        long start = System.currentTimeMillis();
        System.out.println("准备开始");
        startGate.countDown();// 打开开始的门
        endGate.await();// 主线程等待全部结束. (n次countDown结束)
        System.out.println("结束等待");
        long end = System.currentTimeMillis();
        return end - start;

    }

    public static void main(String[] args) throws InterruptedException {
        Runnable task = new Runnable() {
            @Override
            public void run() {
                try {
                    Thread.sleep(1000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println("running task");
            }
        };
        TestLatch testLatch = new TestLatch();
        long during = testLatch.timeTasks(10, task);
        System.out.println("Runtime: " + during);
    }

}
```

# FutureTask<V>\Callable<V>\Runnable
> FutureTask三种状态: waiting,running,completed.
- Runnable
> 顾名思义,有个run函数,可以run.

- Callable<V>
> 比Run多个返回值V.

- FutureTask<V>
> 声明实现的借口是Runnable和Future<V>. 
但实际上里头有个适配器,把Runnable转成Callable.
而且它可以接收Callable作为构造函数的参数.
运行的时候直接ftask.run()即可. 
取结果则是直接ftask.get()即可.

还可以把FutureTask传给Thread的构造函数\加入线程池接受调度.

感觉`FutureTask<V>`挺好用的.

第五章的原则总结:
> 1. 尽量使用final;
2. 不可变对象一定线程安全;
尽量使用不可变对象(空间换时间);
3. 尽量封装,把数据封装到对象中,以便以后构造不可变或者同步策略;
4. 每一个可变对象,都要考虑用锁保护;
5. 保护同一个不变性条件的所有变量时,使用同一个锁;
6. 注意在复合操作上加锁;
7. 不武断得认为不需要同步;
8. 明确地指出(注解或注释)每个类是否线程安全.


# 疑问:
1. Collections.unmodifiableMap(xxx)
```java
final xxx;//引用是不可变的,如果是基本类型,就是值.
final xxx = Collections.unmodifiableMap(xxx);
/**不但引用是不可变的,第一级寻址对象也是不可更改的.
调用map.put(xxx,xxx)会抛unSupported异常.
但显然这不是万能的,用户也可以先用get获取到对象,然后再把对象里的数据给改了.(类似于二级寻址)
因此里面如果存的是不可变对象,是可以的,否则还是可以改.
*/
```
2. ConcurrentMap: 一个线程安全的容器. 保证多线程都能访问一级寻址对象.