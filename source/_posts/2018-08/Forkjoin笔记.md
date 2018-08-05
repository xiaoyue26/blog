---
title: Forkjoin笔记
date: 2018-08-05 17:34:38
tags: 
- java
- 并发
categories:
- java
- 并发
---


# ForkJoin框架
forkjoin类似于一个单机版的mapreduce，只是把多节点多进程换成了多线程。
> 分治法(dfs): 把大任务划分成多个子任务,然后单线程执行、合并结果；
mapreduce: 把大任务划分成多个子任务,然后多节点多进程执行、合并结果；
forkjoin:  把大任务划分成多个子任务,然后多线程执行、合并结果。

优化：
`mapreduce`: 可以通过配置开启预测执行，如果有任务算得慢，会启动新的attempt，取算的快的结果，kill跑得慢的attempt；
`forjoin`: 通过双端队列存储每个线程的任务，如果有线程结束得慢，空闲的线程会进行`工作窃取`,从双端队列的尾部拿任务执行（但是不会重复计算同一个任务，这一点与MR不同）。

## 示例使用代码
范围求和：
```java
public class CountTask extends RecursiveTask<Integer> {
    private final static int THREDSHOLD = 2;
    private final int start;
    private final int end;

    public CountTask(int start, int end) {
        this.start = start;
        this.end = end;
    }

    @Override
    protected Integer compute() {
        int sum = 0;
        boolean canCompute = (end - start) <= THREDSHOLD;
        if (canCompute) {
            for (int i = start; i <= end; i++) {
                sum += i;
            }
        } else {
            // 任务太大, 分治
            int mid = start + (end - start) / 2;
            CountTask left = new CountTask(start, mid);
            CountTask right = new CountTask(mid + 1, end);
            left.fork();
            right.fork();
            int leftRes = left.join();
            int rightRes = right.join();
            sum = leftRes + rightRes;
        }
        return sum;
    }

    public static void main(String[] args) throws ExecutionException, InterruptedException {
        ForkJoinPool forkJoinPool = new ForkJoinPool();// 可以传入并行度参数, 不传则默认从RunningTime取核数作为并行度
        CountTask task = new CountTask(1, 10);// RecursiveTask=> ForkJoinTask
        Future<Integer> res = forkJoinPool.submit(task);// ForkJoinTask
        System.out.print(res.get());


    }
}

```

## 相关类
可以看到代码里主要是继承`RecursiveTask`定义一个计算任务类，定义分治和合并计算结果的操作，然后交给`ForkJoinPool`进行计算即可。（类似于归并排序）

实际上`forkjoin`框架中涉及到的类大致如下：
```java
// 外部接口：
// 1. 需要返回值：
RecursiveTask => ForkJoinTask implements Future<V>, Serializable

// 2. 不需要返回值：
RecursiveAction => ForkJoinTask implements Future<V>, Serializable

// 3. 管理线程池和工作任务：
ForkJoinPool => extends AbstractExecutorService => implements ExecutorService
```

`ForkJoinPool`中使用的双端队列：
```java
// 内部组件 
@sun.misc.Contended
    static final class WorkQueue {
        // 核心部分: 
        volatile int scanState;    // versioned, <0: inactive; odd:scanning
        int stackPred;             // pool stack (ctl) predecessor
        volatile int qlock;        // 1: locked, < 0: terminate; else 0
        volatile int base;         // 队尾,poll用,会被窃取线程更改。
        int top;                   // 队首,push用,只会被当前线程更改,
                                   // 因此没有volatile
        ForkJoinTask<?>[] array;   // 双端队列
        volatile Thread parker;    // == owner during call to park; else null
        volatile ForkJoinTask<?> currentJoin;  // task being joined in awaitJoin
        volatile ForkJoinTask<?> currentSteal; // mainly used by helpStealer
        // 其他成员省略:
        ... 
    ... 
}
```




