---
title: 如何阅读heap dump文件
date: 2018-12-13 21:44:55
tags:
- java
- jvm 
categories:
- java
- jvm 

---


参考资料: http://www.importnew.com/24393.html

# 下载MAT
https://eclipse.org/mat/downloads.php
注意打开的时候要选打开heap dump，而不是open file，不然会打开失败。

## 术语
### Retained Heap
`Retained: 持有,阻拦`
对象以及它所持有的其它引用（包括直接和间接）所占的总内存。

`tips`
```
如果A,B都持有C对象引用，则C的内存不计入A或B。
```

### Shallow heap
`Shallow: 浅`
对象本身占用内存的大小，不包含其引用的对象。
- 常规对象（非数组）的shallow size有其成员变量的数量和类型决定。
- 数组的shallow size有数组元素的类型（对象类型、基本类型）和数组长度决定
#### `Shallow size`最大一般是`byte[], char[], int[]`

## Action面板的功能
### Histogram
`默认Shallow size排序`
列出每个class的实例数量、大小(内存中的对象统计),可以查看某些对象的具体值:
{% img /images/2018-12/stringvalue.png 800 1200 stringvalue %}

进一步操作: `group by package`:
{% img /images/2018-12/hist_groupby.png 800 1200 hist_groupby %}

### Dominator Tree
`默认Retained Heap排序`
列出最大的对象以及其依赖存活的Object。
`TIPS`: 这里有黄点的是GC Root。
  
### Top Consumers
{% img /images/2018-12/top_consumer.png 800 1200 top_consumer %}
从3个角度列出最占用内存的对象，展现形式有表格和图。
三个角度: class,class loader,包。

### Duplicate Class
列出由多个class loader加载的class。


## 图例
带有黄点的对象: GCRoot（如果是`System class`可以不用管）

## 常用操作
`Path to GC Roots` -> `exclude weak references`: 看看为啥没有回收。

`List Object`->`With incoming reference`: 看看都有谁用了这个对象;
{% img /images/2018-12/with_in.png 800 1200 with_in %}

`List Object`->`With outcoming reference`: 看看里面都有啥。
