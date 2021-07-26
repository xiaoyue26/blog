---
title: metaspace笔记
date: 2021-07-26 11:45:43
tag:
- java
- jvm
- metaspace
categories: 
- java
- jvm

---

# metaspace内容
Klass Metaspace+NoKlass Metaspace: 
{% img /images/2021-07/metaspace.png 800 1200 metaspace %}

# Klass Metaspace
> 如果关闭压缩指针，或者堆大于32G，这块儿会存到NoKlass Metaspace里。

存class元数据，JVM对class的表示。
vtable: 类中的方法（我理解类似虚函数指针，指到NoKlass区域）
itable: 类实现的接口
oop map: 类引用的对象地址

# NoKlass Metaspace
方法相关：
> 方法的字节码
参数信息
局部变量表
异常表

常量池
注解
方法计数器（JIT）




# 相关参数
`-Xnoclassgc`: 让 JVM 在垃圾收集的时候不去卸载类.
尽量别开启这个选项。

`–XX:+CMSClassUnloadingEnabled`: 如果使用CMS，需要开启这个选项。

`CompressedClassSpaceSize`: Klass空间的大小。默认1G,可能要改大。
比如可以CompressedClassSpaceSize设置2G, MaxMetaspaceSize设置3G.(看klass和noklass的比例)



# jstat
主要看MC，MU，CCSC，CCSU：
MC: Klass+NoKlass Metaspace已committed的内存大小/KB
MU: Klass+NoKlass Metaspace已使用的内存大小
CCSC: Klass Metaspace已commit的内存大小/KB
CCSU: Klass Metaspace已使用的内存大小/KB

其他:
M: Klass+NoKlass总共使用率
CCS: CCSU/CCSC
MCMN和CCSMN: 0
MCMX: Klass+NoKlass Metaspace两者总共的reserved的内存大小
CCSMX: Klass Metaspace reserved的内存大小

# 生命周期
每个class loader都会有自己的metaspace，各自分配（metachunk,metablock），但是总和是公用的。

普通类metaspace: 基本不太可能卸载，因为需要等class loader卸载；
匿名类metaspace: 跟着匿名类的生命周期。lambda和method handler可以提前卸载，不等待class loader。

卸载以后内存是否会归还：
1. Klass空间必不归还；
2. noKlass空间: 如果Node分配给同一个class loader，比较容易归还；
如果分配给了多个class loader，不释放空间。
一个Node多个chunk；
单个chunk整体属于一个class loader；
一个chunk多个block，每个block是最小分配单元（分配给调用者）。




# 参考
https://javadoop.com/post/metaspace
https://segmentfault.com/a/1190000023235677
https://www.infoq.cn/article/troubleshooting-java-memory-issues