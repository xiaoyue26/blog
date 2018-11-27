---
title: jvisualvm-运行时java程序监控
date: 2018-11-27 12:01:57
tags:
- java
- jvm
categories:
- java
- jvm

---

# 查看进程id
首先需要知道我们关心的java程序的进程id，才能方便后续的操作。
`jps -lvm`
如果不是java程序,则可以用命令`ps aux | grep [name]`。

# 查看进程运行参数
`jinfo -flags <PID>`

# 查看进程各项统计信息
类加载统计:
`jstat -class <PID> `
此外还有:   编译统计、垃圾回收统计、堆内存统计、新生代垃圾回收统计、新生代内存统计等等统计信息。
参考: https://www.cnblogs.com/yjd_hycf_space/p/7755633.html
https://blog.csdn.net/zhaozheng7758/article/details/8623549

缺点: 都是文本、英文,不一定马上看得懂。

# 生成3种dump文件
为了了解更详细进程信息,可以生成3种dump文件:(然后用各种(图形)工具辅助查看)
1. Thread Dump: 线程运行状态;
2. Heap Dump: 堆内存状态;
3. Core Dump: 进程地址空间的内容以及有关进程状态的其他信息写出的一个磁盘文件。

## Thread Dump
拿到进程id以后,我们可以用`jstack`命令生成`Thread Dump`文件。
```sh
jstack <PID>  
```
然后可以:
1. 把Thread dump文件装载入`jvisualvm`查看。
2. 直接文本编辑器打开看;
3. 粘贴到网站,辅助查看: http://spotify.github.io/threaddump-analyzer/

>`jstack`是jvm原生支持的,底层实现原理:
1. `tool.jar`中提供的一个`Attach Listener`,监听相关socket连接。
2. `sa-jdi.jar`中的监听。

## Heap Dump
还可以通过`jmap`命令+进程id，输出`Heap Dump`。
```sh
jmap -histo:live <PID>
```
输出到文件:
```sh
jmap -dump:live,format=b,file=heap.dump <PID>
```
得到`heap dump`文件以后,可以:
1. 用jhat命令: `jhat -port 443 [filename] `,然后访问`http://localhost:443`查看;
2. 装入到`jvisualvm`中查看.

参考: http://www.importnew.com/27804.html


## Core Dump
还可以输出进程的核心转储文件,方法是对进程发送一些会产生`core dump`的信号,例如:
`kill -3 [PID]`
用了上述命令以后`core dump`数据会输出到进程的标准输出中。（可以事先重定向stdout）

详细的信号包括:

|Signal |    Value  |   Action |  Comment|
| ------- | ------:| :------:| :-------------:|
|SIGHUP    |    1  |     Term |   Hangup detected on controlling terminal or death of controlling process
|SIGINT       | 2     |  Term |   Interrupt from keyboard
|SIGQUIT   |    3     |  Core  | Quit from keyboard
|SIGILL     |  4    |   Core   | Illegal Instruction
   |SIGABRT    |   6    |   Core  |  Abort signal from abort(3)
   |SIGFPE     |  8   |   Core  |  Floating point exception
   |SIGKILL     |  9    |   Term  |  Kill signal
   |SIGSEGV    |  11   |    Core  |  Invalid memory reference
   |SIGPIPE   |   13    |   Term  |  Broken pipe: write to pipe with no readers
   |SIGALRM    |  14   |    Term  |  Timer signal from alarm(2)
   |SIGTERM   |   15     |  Term  |  Termination signal
   |SIGUSR1  | 30,10,16  |  Term |   User-defined signal 1
   |SIGUSR2  | 31,12,17  |  Term  |  User-defined signal 2
   |SIGCHLD  | 20,17,18  |  Ign  |  Child stopped or terminated
   |SIGCONT |  19,18,25 |    Cont |   Continue if stopped
   |SIGSTOP  | 17,19,23  |  Stop  |  Stop process
   |SIGTSTP  | 18,20,24 |   Stop  |  Stop typed at tty
   |SIGTTIN  | 21,21,26 |   Stop   | tty input for background process
   |SIGTTOU  | 22,22,27  |  Stop  |  tty output for background process

# jvisualvm
上述几个工具比较零散,而且大部分是文本,不能直观迅速感知内存是上升还是下降.
可以直接用`jvisualvm`命令启动图形界面，直接监控本地的java进程(包括堆、thread dump信息等，还能进行操作)。
{% img /images/2018-11/jvi1.png 800 1200 jvi1 %}
{% img /images/2018-11/jvi2.png 800 1200 jvi2 %}
{% img /images/2018-11/jvi3.png 800 1200 jvi3 %}
{% img /images/2018-11/jvi4.png 800 1200 jvi4 %}
用法:
1. 监控本地java进程;
2. 装载heap dump,thread dump文件查看;
3. 远端启动`jstatd`/`jmx`,打开端口权限,远程监控java程序。

## jstatd
准备配置文件:
```sh
vi jstatd.all.policy 
```
内容如下:
```
grant codebase "file:/usr/java/default/lib/tools.jar" {   
    permission java.security.AllPermission;   
};
```
启动:
```
jstatd -J-Djava.security.policy=./jstatd.all.policy
```
默认会启动在1099端口。
启动以后`jvisualvm`就能创建远程主机，并监控它所有的java程序了。

## jmx
`jstatd`和`jmx`似乎不能同时用，不然`jvisualvm`会卡死.
需要在启动java的时候就指定相应的端口等信息:
```
nohup java \
-Dcom.sun.management.jmxremote=true \
-Dcom.sun.management.jmxremote.port=8080 \
-Dcom.sun.management.jmxremote.ssl=false \
-Dcom.sun.management.jmxremote.authenticate=false \
-Djava.rmi.server.hostname=10.65.89.102 \
-XX:+HeapDumpOnOutOfMemoryError \
-XX:HeapDumpPath=/data/mengqifeng/test \
-classpath pipe-realtime-client-0.0.1-jar-with-dependencies.jar \
com.tencent.kandian.HippoConsumer >> con.log & echo $! > con.pid
```
然后在`jvisualvm`中创建远程`JMX`连接。