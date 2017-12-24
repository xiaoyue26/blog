title: jdk1.7中fork-join并发框架源码浅析及具体实例分析
date: 2016-02-26 19:55:46
tags: java
categories:
- java
---


&emsp;&emsp;java7中的`fork-join`的基本思想就是创立线程池然后试图进行`Work Stealing`:
>  {% img /images/workStealing.png 200 100 Work Stealing %}
  
&emsp;&emsp;所以任务的创建和线程的触发顺序就显得比较微妙而需要额外小心线程同步的问题，若子任务无返回值，尽量使用`invorkAll`方法。也可以参考源码中`invokeAll`的使用方法学习`fork`、`invork`和`join`的使用规范:
{% codeblock invorkAll http://xiaoyue26.github.io/ java.util.concurrent.ForkJoinTask.java %}
 /**
     * Forks the given tasks, returning when {@code isDone} holds for
     * each task or an (unchecked) exception is encountered, in which
     * case the exception is rethrown. If more than one task
     * encounters an exception, then this method throws any one of
     * these exceptions. If any task encounters an exception, others
     * may be cancelled. However, the execution status of individual
     * tasks is not guaranteed upon exceptional return. The status of
     * each task may be obtained using {@link #getException()} and
     * related methods to check if they have been cancelled, completed
     * normally or exceptionally, or left unprocessed.
     *
     * @param tasks the tasks
     * @throws NullPointerException if any task is null
     */
public static void invokeAll(ForkJoinTask<?>... tasks) {
    Throwable ex = null;
    int last = tasks.length - 1;
    for (int i = last; i >= 0; --i) {//从last到1依次fork,而0则直接invork。
        ForkJoinTask<?> t = tasks[i];
        if (t == null) {
            if (ex == null)
                ex = new NullPointerException();
        }
        else if (i != 0)//不为0则fork
            t.fork();
        else if (t.doInvoke() < NORMAL && ex == null)//为0则invork
            ex = t.getException();
    }
    for (int i = 1; i <= last; ++i) {//注意此处是从1开始, 依次join
        ForkJoinTask<?> t = tasks[i];
        if (t != null) {
            if (ex != null)
                t.cancel(false);
            else if (t.doJoin() < NORMAL && ex == null)
                ex = t.getException();
        }
    }
    if (ex != null)
        UNSAFE.throwException(ex);
}
{% endcodeblock %}
输入为两个任务的源码例子:
```java
   /**
     * Forks the given tasks, returning when {@code isDone} holds for
     * each task or an (unchecked) exception is encountered, in which
     * case the exception is rethrown. If more than one task
     * encounters an exception, then this method throws any one of
     * these exceptions. If any task encounters an exception, the
     * other may be cancelled. However, the execution status of
     * individual tasks is not guaranteed upon exceptional return. The
     * status of each task may be obtained using {@link
     * #getException()} and related methods to check if they have been
     * cancelled, completed normally or exceptionally, or left
     * unprocessed.
     *
     * @param t1 the first task
     * @param t2 the second task
     * @throws NullPointerException if any task is null
     */
    public static void invokeAll(ForkJoinTask<?> t1, ForkJoinTask<?> t2) {
        int s1, s2;
        t2.fork();
        if ((s1 = t1.doInvoke() & DONE_MASK) != NORMAL)
            t1.reportException(s1);
        if ((s2 = t2.doJoin() & DONE_MASK) != NORMAL)
            t2.reportException(s2);
    }
```
两个任务的时候也是先`fork t2->invork t1->join t2`这样的流程。

---

fork操作是把任务放入`workQueue`的尾部(`push`):
```java
public final ForkJoinTask<V> fork() {
        Thread t;
        if ((t = Thread.currentThread()) instanceof ForkJoinWorkerThread)
            ((ForkJoinWorkerThread)t).workQueue.push(this);//放入尾部
        else
            ForkJoinPool.common.externalPush(this);
        return this;
    }
```
join操作:
```java
 /**
     * Implementation for join, get, quietlyJoin. Directly handles
     * only cases of already-completed, external wait, and
     * unfork+exec.  Others are relayed to ForkJoinPool.awaitJoin.
     *
     * @return status upon completion
     */
    private int doJoin() {
        int s; Thread t; ForkJoinWorkerThread wt; ForkJoinPool.WorkQueue w;
        return (s = status) < 0 ? s :
            ((t = Thread.currentThread()) instanceof ForkJoinWorkerThread) ?
            (w = (wt = (ForkJoinWorkerThread)t).workQueue).
            tryUnpush(this) && (s = doExec()) < 0 ? s :
            wt.pool.awaitJoin(w, this, 0L) :
            externalAwaitDone();
    }
```
综上所述，整个标准的调用流程应该是先反序`fork`(除了第一个任务),把任务(this)放入队列尾部，调用第一个任务，再顺序`join`(也除了第一个任务)。

---

下面是一个比较复杂、具体的求`double`数组平方和的过程。恰当的组织可以让运算保持分而治之的理念，而不是仅仅只是递归。
```java

import java.util.concurrent.ForkJoinPool;
import java.util.concurrent.RecursiveAction;

public class Applyer extends RecursiveAction {
	static double sumOfSquares(ForkJoinPool pool, double[] array) {
		int n = array.length;
		Applyer a = new Applyer(array, 0, n, null);
		pool.invoke(a);
		return a.result;
	}

	final double[] array;
	final int lo, hi;
	double result;
	Applyer next; // keeps track of right-hand-side tasks

	Applyer(double[] array, int lo, int hi, Applyer next) {
		this.array = array;
		this.lo = lo;
		this.hi = hi;
		this.next = next;
	}

	double atLeaf(int l, int h) {
		double sum = 0;
		for (int i = l; i < h; ++i) // perform leftmost base step
			sum += array[i] * array[i];
		return sum;
	}

	protected void compute() {
		int l = lo;
		int h = hi;
		Applyer right = null;
		while (h - l > 1 && getSurplusQueuedTaskCount() <= 3) {
			int mid = (l + h) >>> 1;//无符号右移(除以2)(若(l+h)为负数,mid就变成整数了)
			right = new Applyer(array, mid, h, right);
			right.fork();
			h = mid;
		}
		double sum = atLeaf(l, h);
		while (right != null) {
			if (right.tryUnfork()) // directly calculate if not stolen
				sum += right.atLeaf(right.lo, right.hi);
			else {
				right.join();
				sum += right.result;
			}
			right = right.next;
		}
		result = sum;
	}

	public static void main(String[] args) {
		ForkJoinPool forkJoinPool = new ForkJoinPool();
		double[] array = { 1, 2, 3 };
		System.out.println(sumOfSquares(forkJoinPool, array));
	}

}
```

`Applyer`作为一个`RecursiveAction`，从最初的一个`Applyer(array, 0, n, null)`,分裂为多个任务，放置于任务列表中，空闲的工作线程从列表中取出任务运行`compute`方法，计算一个叶节点后， 优先再去寻找右边的叶节点进行计算，若靠右的叶节点们先结束运算，再去任务列表中窃取任务进行计算。

其中`getSurplusQueuedTaskCount`方法用于判断当前过剩的任务数量，例如当工作线程数量为`10`，而划分的任务数量为`14`，则不会有空闲的线程来进行任务窃取，大家都很忙，所以就没必要继续划分任务了。反之，若有空闲的工作线程，而且任务足够大，则应该先把任务划分一下，以方便其他空闲线程窃取自己的任务，加快所有任务的完成。

`compute`函数第一个`while`循环先对任务数量和当前工作线程数量进行权衡，如果过剩的任务数量小于3，则进行二分。一直到任务数量足够了，跳出循环。使用`atLeaf()`函数直接计算`(l,f)`范围内的平方和。
    
计算完当前叶节点的任务后，检查右边叶节点的任务是否被其他线程窃取。若被窃取了则等待其完成，反之则自己计算该任务。然后继续往右遍历任务。

所有任务分布在树的叶节点上，并且以链表的形式串联起来，以便相互窃取任务。(注意到由于线程的工作速度不同，叶节点的深度很可能不是一致的。)



参考:
    
\[1] [http://www.molotang.com/articles/694.html][2]
\[2] [http://www.iteye.com/topic/643724][3] (其中fork和join用法不符合jdk源码使用规范)


  [1]: http://www.molotang.com/wp-content/uploads/2013/08/forkjoin%E9%98%9F%E5%88%97-249x300.png
  [2]: http://www.molotang.com/articles/694.html
  [3]: http://www.iteye.com/topic/643724
