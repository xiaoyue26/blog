title: 使用JMX轮询发现死锁
date: 2015-10-30 10:15:48
tags:
- JMX 
- 线程 
- java
categories:
- java

---


[ThreadMXBean][1]是Java虚拟机线程系统的管理接口，可以使用其中的`findDeadlockedThreads`方法自动发现死锁的线程，jdk1.6下包括:
>1. `owner locks(java.util.concurrent)`引起的死锁；
>2. `monitor locks`（例如，同步块）引起的死锁。
```java
public static class DeadLockDetector extends Thread {
  @Override
  public void run() {
    while(true) {
        ThreadMXBean threadMXBean = ManagementFactory.getThreadMXBean();
        long[] ids = threadMXBean.findDeadlockedThreads();
        if (ids != null) {
          System.exit(127);
      }
      try {
        Thread.sleep(1000);//定期轮询
      } catch (InterruptedException e) { }
    }
  }
}
```

```java
public static void main(String[] args) throws Exception {
  Process process = null;
  int exitcode = 0;
  try {
    while (true) {
        process = Runtime.getRuntime().exec("java -cp . DeadlockedApp");
        exitcode = process.waitFor();
        System.out.println("exit code: " + exitcode);
        if (exitcode == 0)
          break;
    }
  } catch (Exception e) {
    if (process != null) {
       process.destroy();
    }
  }
}
```
然而轮询和检测死锁显然都是相当耗费资源的操作，尽量还是自己做好线程的管理。





  [1]: http://download.oracle.com/technetwork/java/javase/6/docs/zh/api/java/lang/management/ThreadMXBean.html
