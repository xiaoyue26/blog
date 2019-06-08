---
title: spark-sql中的分位数算法
date: 2019-05-26 20:36:51
tags: spark
categories:
- spark
 
---


spark有两个分位数算法:
1. `percentile`: 接受Int,Long,精确计算。底层用OpenHashMap,计数，然后排序key;
2. `percentile_approx`：接受Int,Long,Double,近似计算。用的GK算法。论文参见《Space-efficient Online Computation of Quantile Summaries》(http://dx.doi.org/10.1145/375663.375670)
基本思想是以最小空间完成分位数统计，比如把相邻的1w个数压缩成(平均数,个数)元组。如果空间够用，就不进行这种压缩。（所以如果如果统计90分位数，传入的精度参数至少应为10，如果统计999分位数，传入的精度参数至少为1000，默认精度是10000。）

俩算法和Hive版本的基本是一样的。
区别: 
1. spark的percentile多了一个频次参数,也就是可以接受分阶段聚合；(percentile_approx木有)
2. spark底层用的openHashMap,速度快5倍,内存消耗更少。

## 为啥OpenHashMap性能优于HashMap?
https://github.com/apache/spark/blob/master/core/src/main/scala/org/apache/spark/util/collection/OpenHashMap.scala
https://github.com/apache/spark/blob/master/core/src/main/scala/org/apache/spark/util/collection/OpenHashSet.scala
OpenHashMap为了加快速度，增加了一个假设: 所有数据只插入Key/更新Key，不删除Key。
(这个假设在大数据处理/统计的场景下，大多都是成立的)
有了这个假设它可以去掉拉链表，使用线性探测的开放定址法来实现哈希表。

OpenHashMap底层数据委托给了OpenHashSet，所以本质上是看OpenHashSet为啥快。
`OpenHashSet`用BitSet(位图)来存储在不在集合中(位运算，很快)，另开一个数组存储实际数据：
```scala
protected var _bitset = new BitSet(_capacity)
protected var _data: Array[T] = _
  _data = new Array[T](_capacity)
```
这俩成员始终保持等长，_bitset的下标x位置为1时，_data的下标x位置为中就有实际数据。(手动维持联动)
插入数据时，hash一下key生成pos，看看_bitset中对应位置有没有被占用，有的话就死循环++pos：
```scala
def addWithoutResize(k: T): Int = {
    var pos = hashcode(hasher.hash(k)) & _mask
    var delta = 1
    while (true) {
      if (!_bitset.get(pos)) {
        // This is a new key.
        _data(pos) = k
        _bitset.set(pos)
        _size += 1
        return pos | NONEXISTENCE_MASK
      } else if (_data(pos) == k) {
        // Found an existing key.
        return pos
      } else {
        // quadratic probing with values increase by 1, 2, 3, ...
        pos = (pos + delta) & _mask
        delta += 1
      }
    }
    throw new RuntimeException("Should never reach here.")
  }
```
逻辑很简单，由于假设了不会删除key,线性探测法变得实用。
### 小结一下OpenHashSet快的原因:
1. 内存利用率高: 去掉了8B指针结构，能够创建更大的哈希表，冲突减少；
2. 内存紧凑: 位图操作快，一个内存page就能放下很多位图，8B就能放64个位置，缓存友好(while循环pos++)。


# percentile实现: 
`Percentile.scala`文件:
https://github.com/apache/spark/blob/master/sql/catalyst/src/main/scala/org/apache/spark/sql/catalyst/expressions/aggregate/Percentile.scala
首先看注释:
```
/* Because the number of elements and their partial order cannot be determined in advance.
 * Therefore we have to store all the elements in memory, and so notice that too many elements can
 * cause GC paused and eventually OutOfMemory Errors.
/
```
基本思想是把所有元素保存在内存中。
因此它其实支持两阶段聚合:
`_FUNC_(col, percentage [, frequency])`
可以传入一个参数frequency表示频次.
// 2017-02-07加上的特性，比我写hive版本的分阶段聚合udaf早了10个月。

# percentile_approx实现
代码:
https://github.com/apache/spark/blob/master/sql/catalyst/src/main/scala/org/apache/spark/sql/catalyst/expressions/aggregate/ApproximatePercentile.scala
https://github.com/apache/spark/blob/master/sql/catalyst/src/main/scala/org/apache/spark/sql/catalyst/util/QuantileSummaries.scala
底层委托给`QuantileSummaries`实现的。
主要有俩个成员变量：
```
sample: Array[Stat] : 存放桶，超过1000个桶的时候就压缩（生成新的三元组）；
headSampled: ArrayBuffer[Double]：缓冲区，每次达到5000个，就排序后更新到sample.
```
主要思想是减少空间占用，因此很多排序，spark的实现merge sample的时候甚至都没有管俩sample已经有序了，直接sort了：
```scala
// TODO: could replace full sort by ordered merge, the two lists are known to be sorted already.
 val res = (sampled ++ other.sampled).sortBy(_.value)
  val comp = compressImmut(res, mergeThreshold = 2 * relativeError * count)
  new QuantileSummaries(
    other.compressThreshold, other.relativeError, comp, other.count + count)
```
Stat的定义:
```scala
 /**
   * Statistics from the Greenwald-Khanna paper.
   * @param value the sampled value
   * @param g the minimum rank jump from the previous value's minimum rank
   * @param delta the maximum span of the rank.
   */
  case class Stats(value: Double, g: Int, delta: Int)

```
插入的函数:(每N个数，排序至少1次(merge还有1次)，因此是O(NlogN))
```scala
def insert(x: Double): QuantileSummaries = {
    headSampled += x
    if (headSampled.size >= defaultHeadSize) {
      val result = this.withHeadBufferInserted
      if (result.sampled.length >= compressThreshold) {
        result.compress()
      } else {
        result
      }
    } else {
      this
    }
  }

```

插入数据的其中一个步骤:
```scala
private def withHeadBufferInserted: QuantileSummaries = {
    if (headSampled.isEmpty) {
      return this
    }
    var currentCount = count
    val sorted = headSampled.toArray.sorted
    val newSamples: ArrayBuffer[Stats] = new ArrayBuffer[Stats]()
    // The index of the next element to insert
    var sampleIdx = 0
    // The index of the sample currently being inserted.
    var opsIdx: Int = 0
    while (opsIdx < sorted.length) {
      val currentSample = sorted(opsIdx)
      // Add all the samples before the next observation.
      while (sampleIdx < sampled.length && sampled(sampleIdx).value <= currentSample) {
        newSamples += sampled(sampleIdx)
        sampleIdx += 1
      }

      // If it is the first one to insert, of if it is the last one
      currentCount += 1
      val delta =
        if (newSamples.isEmpty || (sampleIdx == sampled.length && opsIdx == sorted.length - 1)) {
          0
        } else {
          math.floor(2 * relativeError * currentCount).toInt
        }

      val tuple = Stats(currentSample, 1, delta)
      newSamples += tuple
      opsIdx += 1
    }

    // Add all the remaining existing samples
    while (sampleIdx < sampled.length) {
      newSamples += sampled(sampleIdx)
      sampleIdx += 1
    }
    new QuantileSummaries(compressThreshold, relativeError, newSamples.toArray, currentCount)
  }
```

获取结果:O(n)
```scala
// Target rank
    val rank = math.ceil(quantile * count).toInt
    val targetError = math.ceil(relativeError * count)
    // Minimum rank at current sample
    var minRank = 0
    var i = 1
    while (i < sampled.length - 1) {
      val curSample = sampled(i)
      minRank += curSample.g
      val maxRank = minRank + curSample.delta
      if (maxRank - targetError <= rank && rank <= minRank + targetError) {
        return Some(curSample.value)
      }
      i += 1
    }
```
 
# 优化思路
结合yuange在微博/km上分享的思路，用计数器区代替密集数据区的hashmap(其实也是GK算法的精确版)。逼近O(N)复杂度。
// TODO benchmark、优化算法