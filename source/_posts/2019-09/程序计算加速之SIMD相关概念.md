---
title: 程序计算加速之SIMD相关概念
date: 2019-09-01 16:27:04
tags:
- java
- SIMD
categories: 
- java

---

# What: 什么是SIMD
SIMD全称`Single Instruction Multiple Data`，单指令多数据流，是一种采用一个控制器来控制多个处理器，同时对一组数据（又称“数据向量”）中的每一个分别执行相同的操作从而实现空间上的并行性的技术。

也就是说SIMD需要CPU指令集的支持，才能用一条指令就同时并行计算多个数据。当然了，这里同时计算时运用的是同一种运算，比如都是加法。

# WHY: 为什么要使用SIMD
能并行计算肯定是要比串行计算快的。
SIMD是cpu层面的加速，当然还有gpu层面的加速比如cuda编程。
如果需要大量浮点数计算、矩阵计算，比如游戏场景、机器学习场景下都是需要这些加速技术的。

# 不同版本和历史
既然说到cpu，肯定绕不开intel和AMD。
最早是intel推出的，利用了多余的寄存器来加速多媒体运算，后来逐渐标准化：

## 缩写:
```
MMX(可能是MultiMedia eXtension的缩写)
SSE(Streaming SIMD Extensions)
AVX(Advanced Vcetor Extension) : 对SSE的后续扩展，主要分为AVX、AVX2、AVX512三种。在目前常见的机器上，大多只支持到AVX系列。
```
## 版本

```
MMX: intel, Pentium;
SSE: intel, Pentium 3;
SSE2: intel, Pentium 4;
SSE3: intel, Pentium 4;
SSE4: intel, Core 2 Duo; 128位。
SSE5: AMD;
AVX:  intel, 因为SSE5被AMD抢先出了,intel恼羞成怒改名叫AVX了; 支持256位。
AVX2: intel, 加入了整形支持。支持256位。
AVX512: intel, 支持521位。
```

# HOW： 如何使用SIMD
## JAVA中使用
jdk8的话，可以`jinfo -flag <pid>`一下，这三个其实是默认配置(java8):
```
-XX:UseAVX=2
-XX:UseSSE=5
-XX:+UseSSE42Intrinsics
```

可以受益的操作：
加减乘除、乘累加。
所以jvm是默认会对一些代码进行SIMD优化，具体方法是自己构造数组，比如本来只是要统计一个数组的总和，用一个局部变量即可，可以改成用一个长度为8的局部数组（或者16、具体长度需要benchmark才知道最优，要符合cpu的SIMD支持长度），然后在8个位置上分别求和，最后把局部数组求和得到答案，这种代码会比直接求和快1倍。

上述trick自然是非常间接地使用了，直接使用SIMD的库还在开发中：
https://openjdk.java.net/jeps/338
估计要等到java10以后才能用上了。
也有一些国外的scala库(LMS)： https://astojanov.github.io/blog/2017/12/20/scala-simd.html?tdsourcetag=s_pcqq_aiomsg
但不知道靠谱不靠谱。


直接使用的库在C语言中是有的，叫做`SIMD Intrinsics`：
https://software.intel.com/sites/landingpage/IntrinsicsGuide/#techs=MMX,SSE,SSE2,SSE3,SSSE3,SSE4_1,SSE4_2,AVX,AVX2,FMA,AVX_512
比如`_mm_set_ps1`函数。
```
???-ss后缀的操作: 4个操作数中有一个参加运算；
???-ps后缀的操作：4个操作数都参加运算。
```
因此我们可以用JNI调用来使用SIMD。
为了一定程度上减少JNI开销的话，可以使用`CriticalNative`。


# 参考资料
https://www.cnblogs.com/lgxZJ/p/8688430.html