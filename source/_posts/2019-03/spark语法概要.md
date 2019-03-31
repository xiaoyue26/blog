---
title: spark笔记
date: 2019-03-31 19:36:22
tags: spark
categories:
- spark

---

job: action分割
task: shuffle分割
application: 多个job

join\cogroup: 确保Rdd1.partitioner = Rdd2.partitioner = join.partitioner
换句话就是 rdd1的key和rdd2的key，还有join时用的key,三者是相等的。

# 语法概要
官方示例代码库:
http://spark.apache.org/examples.html
此外还有安装目录的`examples`目录。

## 生成rdd
```scala
// (1) 从数组:
data = [1, 2, 3, 4, 5]
distData = sc.parallelize(data)
// (1)从文件:
scala> val distFile = sc.textFile("data.txt")
distFile: org.apache.spark.rdd.RDD[String] 
```

## RDD操作
RDD分为：
1. 普通RDD；
2. PairRDD。
两者能用的函数大不相同，每次操作RDD前需要复核一下到底是普通RDD还是Pair RDD。

方法分为:
1. transform: 变换结构;
2. action: 真正有输出，有动作（特例是forEachPartition这种类似遍历，反函数式编程的）。


## 普通RDD-transform
```scala
// scala用箭头=>:
val lineLengths = lines.map(s => s.length)
val totalLength = lineLengths.reduce((a, b) => a + b)
// python用lambda:
lineLengths = lines.map(lambda s: len(s))
totalLength = lineLengths.reduce(lambda a, b: a + b)
```
也可以传递函数:
```scala
// scala借用object单例:
object MyFunctions {
  def func1(s: String): String = { ... }
}
myRdd.map(MyFunctions.func1)
```
其他函数汇总:(因为不是pair rdd，因此基本都无shuffle)
### 无shuffle,1对1
```scala
map(func)： 
filter(func)：无shuffle
```
### 无shuffle,1对多、多对多
```scala
flatMap(func)	:  每个item可以返回一个seq;
mapPartitions(func)	:  输入迭代器，返回迭代器 
mapPartitionsWithIndex(func):  输入迭代器，返回迭代器
```
### 集合运算
```scala
sample(withReplacement, fraction, seed)	
union(otherDataset)	
intersection(otherDataset)	
distinct([numTasks])): 如果是pairRDD，有shuffle，可以定义并行度。
coalesce(numPartitions)	
repartition(numPartitions)	
repartitionAndSortWithinPartitions(partitioner)	
```
此外还有一个`forEachPartition`，返回值为空，是个遍历的action。
有shuffle的变换一般都可以定义`[numTasks]`，也就是可以定义并行度。


### 闭包
闭包是指 `executor` 要在`RDD`上进行计算时必须对执行节点可见的那些变量和方法。闭包被序列化并被发送到每个 `executor`。

闭包的变量副本(序列化后)发给每个 `executor`.

## Pair RDD-transform
加上了很多shuffle操作的函数(算子)
```scala
val pairs = lines.map(s => (s, 1))
val counts = pairs.reduceByKey((a, b) => a + b)
```
其他函数汇总:
```scala
groupByKey([numTasks])： 返回(K, Iterable<V>)，一般用reduceByKey代替这个算子
reduceByKey(func, [numTasks])	
aggregateByKey(zeroValue)(seqOp, combOp, [numTasks])	
sortByKey([ascending], [numTasks])	
join(otherDataset, [numTasks]): (K, V) 和 (K, W)  =>  (K, (V, W))
cogroup(otherDataset, [numTasks])	
cartesian(otherDataset)	
partitionBy(partitioner): 一般比repartition靠谱，因为下一步能用到key，而不是随机划分
```


Action汇总:
可以注意到凡是最终需要输出、反函数式编程的（遍历）就是action，最后汇聚到driver单点处理。
```scala
reduce(func): 注意和map相反,reduce是action。数据最后聚合成单点。
collect()	
count()	
first()	
take(n)
takeSample(withReplacement, num, [seed])	
takeOrdered(n, [ordering])	
saveAsTextFile(path)	
saveAsSequenceFile(path)	
countByKey()	: 因为整合了reduce的功能
foreach(func) 以及foreachPartition等。
```

### 自定义UDAF的核心: combineByKey
大部分shuffle算子都是调用`combineByKey`实现的，可以说`combineByKey`就是shuffle的核心。
combineByKey定义:
```scala
def combineByKey[C](
  createCombiner: V => C,
  mergeValue: (C, V) => C,
  mergeCombiners: (C, C) => C): RDD[(K, C)] = self.withScope {
combineByKeyWithClassTag(createCombiner, mergeValue, mergeCombiners)(null)
}

```
#### 示例
```scala
val pairs = sc.parallelize(List(("prova", 1), ("ciao", 2),
                                ("prova", 2), ("ciao", 4),
                                ("prova", 3), ("ciao", 6)))
// aggregateByKey:
pairs.aggregateByKey(List[Any]())(
  (aggr, value) => aggr ::: (value :: Nil),
  (aggr1, aggr2) => aggr1 ::: aggr2
).collect().toMap
// combineByKey:
pairs.combineByKey(
  (value) => List(value),
  (aggr: List[Any], value) => aggr ::: (value :: Nil),
  (aggr1: List[Any], aggr2: List[Any]) => aggr1 ::: aggr2
).collect().toMap
```
combineByKey比aggregateByKey更加通用，区别是它的第一个参数创建初始聚合器都是函数，而aggregateByKey第一个参数是一个初始值。




## 缓存
```
cache()
persist([LEVEL])
```
在 shuffle 操作中（例如 reduceByKey），即便是用户没有调用 persist 方法，Spark 也会自动缓存部分中间数据.

## 广播变量
`immutable`
先用action从sc(driver)上广播出去，然后用`.value`访问。
```scala
scala> val broadcastVar = sc.broadcast(Array(1, 2, 3))
broadcastVar: org.apache.spark.broadcast.Broadcast[Array[Int]] = Broadcast(0)

scala> broadcastVar.value
res0: Array[Int] = Array(1, 2, 3)
```


## 累加器
数值型。
可变。
{% img /images/2019-03/acc.jpg 800 1200 acc %}
```scala
scala> val accum = sc.longAccumulator("My Accumulator")
accum: org.apache.spark.util.LongAccumulator = LongAccumulator(id: 0, name: Some(My Accumulator), value: 0)

scala> sc.parallelize(Array(1, 2, 3, 4)).foreach(x => accum.add(x))
...
10/09/29 18:41:08 INFO SparkContext: Tasks finished in 0.317106 s

scala> accum.value
res2: Long = 10
```
自定义累加器需要实现的3个方法:（类似于`combineByKey`）
```scala
reset : 清零。防止重算。
add: 累加
merge: 合并累加器。
```
示例:
```scala
class VectorAccumulatorV2 extends AccumulatorV2[MyVector, MyVector] {

  private val myVector: MyVector = MyVector.createZeroVector

  def reset(): Unit = {
    myVector.reset()
  }

  def add(v: MyVector): Unit = {
    myVector.add(v)
  }
  ...
}
// Then, create an Accumulator of this type:
val myVectorAcc = new VectorAccumulatorV2
// Then, register it into spark context:
sc.register(myVectorAcc, "MyVectorAcc1") // 注册一下
```