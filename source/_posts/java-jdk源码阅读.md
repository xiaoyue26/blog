---
title: java jdk源码阅读
date: 2017-02-11 19:40:34
tags: java
categories: java

---

# Collections
sort,shuffle,等各种集合运算. 
返回并发集合,同步容器,不可修改集合,等等.

# Arrays
fill,等等. 

# Math. StrictMath
addExact
subExact
在加减溢出时可以抛异常.(Math.abs不抛异常)
```
// 位运算小技巧
// 1. x和y异号 <=> 异或后<0 
(1)x^y <0 
// 异号定义: 都不等于0, 异号.

(2)x和y同号 => x^y >=0 
一般使用(1).
// 2. &
(1) x和y均小于0   <=> x&y < 0 
(2) x和y均大于0  =>  x&y>=0
(3) x和y异号 => x&y>=0
// (2),(3)难以区分,一般使用(1) .
```

