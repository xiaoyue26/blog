---
title: spark中RDD，Dataframe，DataSet区别对比
date: 2019-04-29 10:16:54
tags: spark
categories:
- spark

---

# RDD，Dataframe，DataSet的定义
`RDD`: immutable、spark的基础数据集。底层api； 1.0+, 存放java/scala对象;
`Dataframe`: immutable、多了列名、Catalyst优化。高级api;  1.3+, 存放row对象;(有列名)
`Dataset`: 比Dataframe多类型安全。高级api。 1.6+, 存放java/scala对象(暴露更多信息给编译期)
可以理解成DataSet每行存一个大对象。（比如样例中的`DataSet[Person]`）

DataSet一般比DataFrame慢一点点，多了类型安全的开销（`Row`=>`Java类型`）

## 这里说的类型安全是什么？
> 是编译期类型安全

Dataframe: 访问不存在的列名=>运行时报错; （非类型安全）
DataSet:   访问不存在的列名=>编译时报错。 （类型安全）

DataSet的中每个元素都是case class之类的,完全定义类型的,因此编译时就能确定schema合法性。

`Dataset[Row]` = `DataFrame`
因此Row可以看做非编译期类型安全的对象。

## java/scala对象 <=> DataFrame的row对象/DataSet的元素:  
1. 对于dataframe: 传入StructType;
2. 对于dataSet:  使用Encoder。 

类似于hive的反序列化:
### hive反序列化原理:
```
HDFS files –> InputFileFormat –> <key, value> –> Deserializer –> Row object
```
### hive序列化原理(`Serialize`):
```
Row object –> Serializer –> <key, value> –> OutputFileFormat –> HDFS files
```

`Encoder`: `Dataset[Row]` -> `Dataset[T]`
JVM对象和非堆自定义内存二进制数据
Encoders生成二进制代码来跟非堆数据交互，并且提供有需访问给独立的参数而不是对整个对象反序列化。


# 优化
RDD: 没有优化, 程序员自己保证RDD运算是最优的;
DataFrame: 走catalyst编译优化,类似于Sql的优化。根据成本模型，逻辑执行计划优化成物理执行计划。
DataSet: 同DataFrame. 

强调一点,DataFrame底层也是用的RDD实现，因此如果程序员足够牛逼，理论上执行计划能写得比DataFrame的计划好。

# 序列化
shuffle的时候、或者cache写内存、磁盘的时候，需要序列化。

`RDD`: 使用java序列化(或者kryo)成二进制; (成本高)
`DataFrame`: Tungsten计划优化。序列化到堆外内存，然后无须反序列化，直接根据schema操作二进制运算。(因为DataFrame比RDD多一个schema信息)
https://github.com/hustnn/TungstenSecret
`DataSet`: 基本同DataFrame，多了Encoder的概念。访问某列的时候，可以只反序列化那个局部的二进制。

## Tungsten binary format
钨丝计划使用的二进制格式

# 垃圾收集
RDD: 由于上一节中的序列化，gc压力较大；
DataFrame: 放堆外内存，无需jvm对象头开销，无gc；
DataSet: 同df.


# 钨丝计划的秘密：
https://github.com/hustnn/TungstenSecret

总结一下钨丝计划的3大优化：
1. 内存管理、直接操作二进制数据：放堆外内存(避免gc开销)，不用jvm对象(减少对象头开销)。
2. 缓存友好的计算: 考虑存储体系;(flink也有) 对指针排序，根据schema访问key的二进制。(估计key只能是原生类型，其他类就得反序列化了)
3. code generation：充分利用最新的编译期和cpu性能：把jvm对象转到堆外unsafeRow，以便利用第一点。

比如8B的String的对象头有40B开销。
UnsafeShuffleManager： 直接在serialized binary data上sort而不是java objects

## wholeStageCodeGen
把transform的很多步骤，合并成一个步骤，使用字符串插值生成一份重写后的代码(逼近于手写最优解)，然后用Janino(微型运行时嵌入式java编译器)编译成字节码。