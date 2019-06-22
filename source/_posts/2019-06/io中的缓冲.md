---
title: io中的缓冲——如何理解O_Direct
date: 2019-06-22 10:56:52
tags: 
- io
categories:
- java
- Netty 

---

IO缓冲主要有4层:
1.用户自己的缓冲;
2.库缓冲;
3.内核缓冲;
4.磁盘缓冲。


{% img /images/2019-06/io_layer.gif 400 600 io_layer %}
=== 应用层: (进程挂丢数据) 看到文件句柄
`application buffer`: 比如我们代码中写的int[]arrayData;
`clib buffer`: `fwrite`以后到这层。这里写的是c库(`IObuffer`)，也可能是java库中的缓冲(`BufferedOutputStream`)。
如果数据才到这一层库缓冲，还没系统调用，此时程序core dump的话，数据就丢了。
=== 内核层: (内核挂丢数据)  看到inode和数据块
`page cache`: 内核层的缓冲。fflush以后到这里, fclose先到这里然后继续到磁盘。
`driver`: 具体设备的驱动软件
=== 设备层: （断电丢数据） 看到扇区
`disk cache`: 磁盘缓冲。fsync/fclose至少到这里。fsync是同步会完全等返回。


## 为啥要有库缓冲
（比如`clib buffer`）
因为从应用层到内核层需要系统调用、内核态切换，开销比较大，为了减少这件事发生的次数，没有必要因为1个字节的改动发生系统调用。

### 绕过库缓冲的方案: 
用mmap(内存映射文件),把内核空间的page cache映射到用户空间。

## 为啥要有内核缓冲
内核用pdflush线程循环检测脏页,判断是否写回磁盘。
由于磁盘是单向旋转,重新排布写操作顺序可以减少旋转次数。(合并写入)
plus:
`O_SYNC`参数: 访问内核缓冲时是异步还是同步。`O_SYNC`表示同步。

### 绕过内核缓冲的方案
用`O_Direct`参数，直接怼Disk cache。

## 为啥要有磁盘缓冲
驱动通过DMA，将数据写入磁盘cache。
磁盘缓冲主要是保护磁盘不被cpu写坏。是与外部总线交换数据的场所。（断电丢数据）

#### 绕过磁盘缓冲的方案
用`RAW设备写`,直接写扇区: fdisk,dd,cpio工具。




