title: Singleton 单例模式
date: 2015-10-05 17:02:26
tags:
- java 
- 设计模式
categories:
- java
---

网上说得很清楚了：
[Singleton链接](http://blog.csdn.net/haoel/article/details/4028232)
`CmdInvokeSingleton.java`
```java
public enum CmdInvokeSingleton {
    INSTANCE {
        /**
         * 运行shell脚本
         * 
         * @param shell 需要运行的shell脚本
         */
        @Override
        protected void execShell(String shell) {
            try {
                Runtime rt = Runtime.getRuntime();
                final Process process = rt.exec(shell);
                DaemonThread daemonThread = new DaemonThread(process);
                daemonThread.start();//Destroy process if necessary
                int exitcode = process.waitFor();
                daemonThread.interrupt();
                System.out.println("finish:" + exitcode);
            } catch (Exception e) {
                e.printStackTrace();
            }
        }

    };
    protected abstract void execShell(String shell);
}

```
---
`DaemonThread.java`
```java
public class DaemonThread extends Thread {
    /**
     * 监视指定的进程，3秒后强制销毁它。
     * */
    private Process process;
    public DaemonThread(Process process) {
        this.process = process;
    }
    public void run() {
        try {
            sleep(3000); // 延迟3秒
        } catch (InterruptedException e) {
            System.out.println(e.getMessage());
        }
        process.destroy();
    }
}

```
---
调用的时候：
```java
 try {
            String cmdString = "ls > out.txt";
            CmdInvokeSingleton.INSTANCE.execShell(cmdString);
        } catch (Exception e) {
            e.printStackTrace();
            return "invoke failed.";
        }
```
---
下一步可以学习cakephp。
