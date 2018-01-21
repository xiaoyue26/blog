---
title: java并发编程的艺术笔记-第三章
date: 2018-01-21 22:10:41
tags: 
- java
- 并发
categories:
- java
- 并发
---


# Java内存模型
内容有4个部分:
1. 基本概念
2. 顺序一致性
3. 同步原语的内存语义
4. 内存模型的设计原理

## 3.1 基本概念
并发编程的两个关键问题:
1. 线程之间通信: 何种机制交换数据,包括共享内存和消息传递;
2. 线程之间同步: 不同线程的操作发生的顺序. 

JAVA的线程通信: 共享内存(隐式). 

JAVA的内存:
1. 堆内存(共享):   实例域,静态域,数组元素;
2. 栈内存(不共享): 局部变量,方法参数,异常参数.

线程A,B通信流程:
1. 线程A把自己内存中更新过的共享变量刷新到主内存中;(把cpu缓存刷到内存)
2. 线程B把到主内存中读取A更新后的共享变量. (把内存刷新到cpu缓存)

## 3.1.3 源代码到指令序列的重排序
3种重排类型:
1. 编译器优化的重排序: 单线程程序语义不改变;
2. 指令级并行的重排序: 多线程的指令重排并行执行;
3. 内存系统的重排序: 指令实际执行的时候,由于缓存/缓冲的存在,加载和存储可能看起来是乱序执行.

2和3属于处理器重排序.

JAVA内存模型约束:
1. 禁止某些编译器优化;
2. 插入某些特定类型内存屏障(Memory Fence指令),禁止特定类型处理器重排序.

## 3.1.4 并发编程模型的分类
写缓冲区: 临时保存向内存写入的数据. (避免cpu等待io)
仅对该cpu可见. 

内存屏障指令:
**1.LoadLoad Barries**
```
Load1;LoadLoad;Load2
// 确保Load1的装载先于Load2及后续Load指令
```

**2.StoreStore Barries:**
```
Store1;StoreStore;Store2
// 确保Store1的数据先于Store2及后续Store指令
// 刷新到内存
```

**3.LoadStore Barries:**
```
Load1;LoadStore;Store2
// 确保Load1的装载先于Store2及后续Store指令
```

**4.StoreLoad Barries:**
```
Store1;StoreLoad;Load2
// 确保Store1的装载先于Load2及后续Load指令
// 刷新到内存.
// 且会使它之前的所有内存访问指令(Load和Store)都完成后,才执行之后的.
```

其中第四个,StoreLoad屏障最严格,会把写缓冲区刷新到内存,开销最大,大部分cpu都支持.



### 3.1.5 Happens-before规则
1. 程序顺序规则: 单线程内顺序;
2. 锁规则:  解锁先于获得锁;
3. volidate: 写before读;
4. 传递性: 上述规则可以传递.

上述规则被JVM翻译成各种约束,影响了编译器重排序和处理器重排序.

### 3.2.1 数据依赖性
编译器和处理器: 
1. 单线程: 考虑数据依赖性; (写后读,写后写,读后写)
2. 不同线程\不同处理器: 不考虑数据依赖性.

### 3.2.2 as-if-serial语义
编译器和处理器: 单线程语义不变.

### 顺序一致性
顺序一致性: 所有线程看到的执行顺序一致.
正确使用了同步机制,才能达到顺序一致性.

### volatile变量
读: 读取最新的内存值(不一定是CPU缓存值); (有原子性)
写: 写入cpu缓存后,刷新到内存; (有原子性)
自增: 相当于先读后写两个操作, 不具有原子性. 

用`volatile`代替锁.(从内存语义上说,由于它与锁有相同内存效果,可以实现)
```
volatile boolean flag=false;
A:
a=10;
flag=true;

B:
int i;
while(true){
if(flag){
i=a;
break;
}
}
```

1. 对A来说,由于单线程的语义符合程序顺序规则,flag=true发生在a=10之后;
2. 对B来说,由于flag是volatile变量,flag=true之后才能取到i. 因此达到了通信的效果,B中能正确取到A中的a的10.

### 3.4.4 volatile内存语义的实现
内存屏障:(JMM内存模型采用的是保守策略,保证在所有cpu上都能正确)
1. volatile写前面插入StoreStore屏障;
2. volatile写后面插入StoreLoad屏障;
3. volatile读后面插入LoadLoad屏障;
4. volatile读后面插入LoadStore屏障.

volatile写指令序列:
```
普通读写
StoreStore屏障
volatile写
StoreLoad屏障
```

volatile读指令序列:
```
volatile读
LoadLoad屏障
LoadStore屏障
普通读写
```

## 3.5 锁的内存语义
`ReentrantLock`的实现依赖于:
```
FariSync
NonfaiSync
Sync
AQS(AbstrackQueuedSynchronizer)
```
其中AQS里头有个`volatile`变量:
```
private volatile int state;
```
锁的内存语义基本就靠这个`volatile`变量和CAS操作.

1.加锁流程:(公平锁)
```
获取volatile变量state;
CAS操作更改state变量的值.
若更改成功则获取锁成功.
```
2.解锁流程:(公平锁)
```
一些操作...
写state变量. 
其他线程立刻发现了state变化,可以竞争这个锁了.
```


CAS操作能保证(读-改-写)操作原子执行.
CAS操作的底层实现:
```
LOCK总线;
禁止该指令与前后读写指令重排;
将写缓冲区数据刷新到内存.
```

### 3.5.4 concurrent包的实现
实现层次如下:
```
LOCK,同步器,阻塞队列,Executor,并发容器
AQS,非阻塞数据结构,原子变量类
volatile变量的读写,CAS
```

## 3.6 final域的内存语义

### 3.6.1 final域的重排序规则
1. 在构造函数内对一个final域的写入,与随和把这个对象的引用赋值给一个引用变量,这俩操作不能重排;
2. 初次读包含final域的对象,初次读这个final域,这俩操作不能重排.

规则1的实现(构造函数中):
```
final域的写
StoreStore屏障
构造函数return 
```

示例代码:
```
FinalObj obj=inputObj; // 别的线程负责创建的对象
int a=obj.i;// 如果i是final的,那一定能读到初始化以后的值; 
int b=obj.j;// 如果j不是final的,那可能读到还没初始化的值.
```

规则2的实现:
```
LoadLoad屏障
final域的读
```
大多数cpu不需要这个规则也能正确进行final读,但还是有少部分cpu会无视间接依赖,因此需要这个规则.(需要加入这个屏障)
示例代码:
```
FinalObj obj=inputObj; // 别的线程负责创建的对象
int a=obj.i;// 如果i是final的,那会等obj读到对象引用后再去读i;
int b=obj.j;// 如果j不是final的,那可能没等到读到对象obj,就直接取读j了,读取错误.
```


最后,由于x86处理器很多重排并不会做,所以其实final域的内存屏障都不需要加,会被省略.