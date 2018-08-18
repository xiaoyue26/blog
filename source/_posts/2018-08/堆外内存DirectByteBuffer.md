---
title: 堆外内存DirectByteBuffer
date: 2018-08-18 20:52:15
tags: 
- java
categories:
- java
---


堆外内存: 不进行GC，防止JNI访问错位。

## 使用方式
使用堆外内存的两种方式：
1.隐式：比如读写文件时：
> 
读文件: 文件(disk)=>堆外内存=>堆内内存;(过程中JVM保证不GC)
写文件: 堆内内存=>堆外内存=>文件(disk)。

2.显式：使用`DirectByteBuffer`，直接在堆外分配空间，节省1倍空间，减去一倍拷贝操作。


## 使用场景
(因为不gc)
生命期较长的大对象;
创建次数不会太多。
// 如直接的文件拷贝操作。

## 优点
1.对于大内存有良好的伸缩性
2.对垃圾回收停顿的改善可以明显感觉到
3.在进程间可以共享，减少虚拟机间的复制

## 配置
```java
-XX:MaxDirectMemorySize
-Dsun.nio.MaxDirectMemorySize
directMemory = Runtime.getRuntime().maxMemory()
```

## 堆外内存回收
三种方法:
1.达到限制触发自动回收;(`system.gc()`) // 可能被配置`-XX:+DisableExplicitGC`关闭。

2.手动调用`Unsafe`的`freeMemory`接口。

3.使用`DirectByteBuffer`，它在初始化的时候会创建`Cleaner`这个Cleaner对象会在合适的时候执行`unsafe.freeMemory(address)`，从而回收这块堆外内存。
{% img /images/2018-08/collect.png 400 600 堆外内存回收 %}

4.手动调用`((DirectBuffer)bb).cleaner().clean();`.
// 内部还是调用`System.gc()`,所以一定不要`-XX:+DisableExplicitGC`。

 

相关代码:
```java
private static class Deallocator implements Runnable  {
    private static Unsafe unsafe = Unsafe.getUnsafe();
    private long address;
    private long size;
    private int capacity;
    private Deallocator(long address, long size, int capacity) {
        assert (address != 0);
        this.address = address;
        this.size = size;
        this.capacity = capacity;
    }
 
    public void run() {
        if (address == 0) {
            // Paranoia
            return;
        }
        unsafe.freeMemory(address);
        address = 0;
        Bits.unreserveMemory(size, capacity);
    }
}
```