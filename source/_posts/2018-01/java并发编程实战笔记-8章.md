---
title: java并发编程实战笔记-8章
date: 2018-01-01 15:34:18
tags: java
categories:
- java
- 并发
---

# 线程池的使用
这章介绍实际应用中配置调优的一些高级选项. 以及各种坑.
# 8.1  任务与执行策略的隐形耦合
有些任务需要指定执行策略:
1. 依赖性任务: 就是任务之间不独立
2. 线程不封闭的: 就是只能单线程跑的.
3. 对响应时间敏感的: 如GUI. 
4. 使用ThreadLocal的: 线程池会重用线程. 因此可能有风险.

## 8.1.1 死锁
有界线程池不能无限提交. 如果里头的任务都死锁了,线程池也死锁了.

## 8.1.2 响应时间
如果任务都很慢,线程池的响应时间自然也慢.
可以限时或者增大线程池容量.

# 8.2 线程池大小公式
N = cpu数量 = Runtime.getRuntime().availableProcessors();
U = 目标cpu利用率
W/C= 等待时间和执行时间的比率 (响应度)
SIZE = N*U/(1+W/C) 


# 8.3 配置ThreadPoolExecutor
可以通过Executors获取jdk设计好的一些线程池实现.
## 8.3.1 线程的创建与取消
基本大小: 没有任务时候的线程大小. 
最大大小: 上限.
存活时间: 线程空闲时间达到存活时间,则被回收.

## 8.3.2 管理队列任务
线程池满了以后,提交的任务进入等待队列.
newFixedThreadPool: 无界等待队列 LinkedBlockingQueue
newSingleThreadExecutor: 无界等待队列 LinkedBlockingQueue

有界等待队列的话,需要饱和策略. 

## 8.3.3 饱和策略
1. 中止(默认): abort. 抛异常.
2. 调用者运行: 让主线程自己干. 拥塞会外延到TCP层. 
3. 丢弃: 抛弃该任务. 不抛异常.
4. 丢弃最老: 丢弃下一个将要执行的.(如果用了优先级队列,就是抛弃优先级最高的,会造成错误.)


# 8.4 扩展ThreadPoolExecutor
需要实际需求和应用案例才能学会. 

# 8.5 递归算法的并行化
首先循环可以并行化:
```
forl(final Ele e: eles){
    exec.execute(
    new Runnable(){
      public void run(){process(e);}
    }
    );
}
```
递归也一样, 遍历依然是递归的, 但把每一个节点的计算收集到线程池中,异步计算.
```
dfs(node,exec,results){
 exec.execute(...);
 dfs(node.children());
}

```


第九章是图形界面,略过.

