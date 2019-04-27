---
title: spark中的encoder
date: 2019-04-27 11:23:26
tags: spark
categories:
- spark

---


参考资料:
https://stackoverflow.com/questions/53949497/why-a-encoder-is-needed-for-creating-dataset-in-spark
`It also uses less memory than Kryo/Java serialization.`


# What: Encoder是啥?
所有`DataSet`都需要`Encoder`。

`Encoder`是spark-sql用来序列化/反序列化的一个类。主要用于`DataSet`。
本质上每次调用`toDS()`函数的时候都调用了`Encoder`，不过有时候我们察觉不到，因为用了隐式调用(`import spark.implicits._`)。
可以直接看`Encoder`源码注释中的样例:
```
import spark.implicits._
val ds = Seq(1, 2, 3).toDS() // implicitly provided (spark.implicits.newIntEncoder)
```
将数字(JVM对象)转换为DataSet中的元素。(类似于Hive中的序列化,把line转换为row)
这里由于是常见的原始类型，所以spark提供了隐式encoder的调用，隐藏了这些细节。

# Why: 为啥用Encoder?
stackoverflow上说encoder消耗更少的内存。因为kryo把`dataSet`中的所有行都变成了一个打平的二进制对象。
`10x faster than Kryo serialization (Java serialization orders of magnitude slower)`

# When: 啥时候使用Encoder
`Encoder`适用于原始类型、case class对象(因为有默认的apply/unapply方法)、spark-sql类型。

# How: 怎么使用Encoder:
## 1. 创建DataSet时显式使用:
源码注释中的样例:
```java
// eg1: String:
List<String> data = Arrays.asList("abc", "abc", "xyz");
Dataset<String> ds = context.createDataset(data, Encoders.STRING());
// eg2: 复合Tuple:
Encoder<Tuple2<Integer, String>> encoder2 = Encoders.tuple(Encoders.INT(), Encoders.STRING());
List<Tuple2<Integer, String>> data2 = Arrays.asList(new scala.Tuple2(1, "a");
Dataset<Tuple2<Integer, String>> ds2 = context.createDataset(data2, encoder2);
```
## 2. 创建DataSet时隐式使用:
看createDataset的签名:
```
def createDataset[T](data: RDD[T])(implicit arg0: Encoder[T]): Dataset[T]
// Creates a Dataset from an RDD of a given type.
```
所以Encoder其实可以隐式传:
```
import spark.implicits._
val ds = Seq(1, 2, 3).toDS() // implicitly provided
```
## 3. UDAF中使用:
https://github.com/apache/spark/blob/master/examples/src/main/scala/org/apache/spark/examples/sql/UserDefinedTypedAggregation.scala

当我们为`DataSet`定义`UDAF`的使用。
语义上: 因为涉及到数据转换，不可避免地会需要使用`Encoder`，这个时候是显式使用。
语法上: 由于继承了`Aggregator`也必须使用`Encoder`。