title: java线程同步之callable与future
date: 2015-10-14 21:11:25
tags:
- java
categories:
- java

---


 - 背景：
> 多线程编程中，若使用`Thread`或者`Runnable`则无法便捷地获取线程的执行结果；此外在线程同步时，处理超时任务时可以省得自己创建守护线程，所以应该使用`Callable`和`Future`,([Callable和Future的基本描述][1])这样也能方便地配合`ExecutorService`使用。

 - 基本用法:
```java
Future<String> future=DbThreadPool.INSTANCE.add(daemonThread);
//1秒内没得到结果就不再等待:
future.get(1000, TimeUnit.MILLISECONDS);
/*取消Callable任务,不管有没在执行：*/
future.cancel(true);
/**假如还没在执行,取消Callable任务：*/
future.cancel(false);
```

---

这样线程同步就可以用语言特性方便地解决了。然而实际项目使用`tesseract`时，调用的是linux命令,所以创建的是进程,并没有在jdk中找到直接的语言特性支持来解决子进程的超时问题,还是得手动创建守护线程解决进程和线程的同步问题。

> 其实也可以使用`tess4J`来调用`tesseract`,避免自己创建进程,但是经过测试,`tess4J`的效率低得惊人，而且包比较大，所以放弃使用它。

**完整的代码:**
1. 首先创建线程池:
```java
public enum DbThreadPool {
    INSTANCE{
    int cpuNums = Runtime.getRuntime().availableProcessors();
    int poolSize = 10;
    int numThreads = cpuNums * poolSize;
    ExecutorService executorService = new ThreadPoolExecutor(numThreads, numThreads, 0L,
    TimeUnit.MILLISECONDS, new LinkedBlockingQueue<Runnable>());
     
    @Override
    public <M> Future<M> add(Callable<M> task) {
        return executorService.submit(task);
    }
    @Override
    public void add(Runnable task) {
        executorService.execute(task);
    }
    @Override
    protected void shutdown() {
        //等待任务结束后关闭线程池.
        executorService.shutdown();
    }
    @Override
    protected List<Runnable> shutdownNow() {
        //不等待任务结束,直接关闭线程池.
        return executorService.shutdownNow();
    }
    
    };
    protected abstract <M> Future<M>  add(Callable<M> task);

    protected abstract void add(Runnable task);
    protected abstract void shutdown();
    protected abstract List<Runnable> shutdownNow();
}
```
2. 守护线程:
```java
public class DaemonThread implements Callable<String> {
    private Process process;
    public DaemonThread(Process process) {
        this.process = process;
    }
    @Override
    public String call() throws Exception {
        try {
            System.out.println("睡了！");
            Thread.sleep(500); // 延迟0.5秒
            System.out.println("醒了！");
        } catch (InterruptedException e) {
            System.out.println(e.getMessage());
        }
        System.out.println("操作前");
        process.destroy();
        System.out.println("操作后");
        return "Success";
    }
}
```

3. 脚本调用类,其实这个也没必要用单例,只是自己想多写几次练熟一点:
`synchronized`也不是必要的,只是实际使用时希望控制创建的子进程数量, 不希望创建太多子进程,而是维持在1个子进程。
```java
import java.util.concurrent.Future;

public enum CmdInvokeSingleton {
    INSTANCE {
        /**
         * 运行shell脚本
         * @param shell 需要运行的shell脚本
         */
        @Override
        synchronized protected void execShell(String shell) {
            try {
                Runtime rt = Runtime.getRuntime();
                final Process process = rt.exec(shell);
                DaemonThread daemonThread = new DaemonThread(process);
                Future<String> future=DbThreadPool.INSTANCE.add(daemonThread);
                int exitcode = process.waitFor();
                System.out.println("exitcode:" + exitcode);
                future.cancel(true);
            } catch (Exception e) {
                e.printStackTrace();
            }
        }

    };
    protected abstract void execShell(String shell);
}
```

 

  [1]: http://www.cnblogs.com/dolphin0520/p/3949310.html


