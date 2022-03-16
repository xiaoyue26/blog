---
title: cms的gc触发条件
date: 2022-03-16 10:14:56
tag:
- jvm
- cms
- gc
categories: 
- java
- jvm

---

cms的gc分为:
- 前台收集
- 后台收集

## 前台收集
触发条件：对象分配，空间不够；
算法： 标记清除（有碎片）

## 后台收集
定时任务扫描：(间隔=`CMSWaitDuration`=2s)
1. 有显式调用（`System.gc()`）;
2. 预测要满了(根据历史统计数据，且未配置`UseCMSInitiatingOccupancyOnly`)；
（第一次的话50%老年代占用就开始gc了）
3. 老年代占用>阈值;
4. `Young GC`可能失败或已失败;(没空间晋升了)
5. `metaspace`扩容前，会进行一次cms gc；


## 压缩gc(full gc)
触发条件: 前台收集之前，cms可能选择进行一次压缩gc(`full gc`)；(yong+old+metaspace)

触发条件:
(1)gc次数(前台收集+压缩`full gc`) > `CMSFullGCsBeforeCompaction`；(默认0，每次都full gc)
(2)`System.gc()`，则触发；
(3)young gc可能失败或已经失败(晋升失败),则认为碎片太多，需要full gc压缩一下；
(4)配置了`CMSCompactWhenClearAllSoftRefs`, 则需要在内存不够时清理软引用，所以需要full gc;


减少这4种情况的触发，则可以减少full gc，提高性能。

