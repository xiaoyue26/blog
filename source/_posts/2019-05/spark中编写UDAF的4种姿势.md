---
title: spark中编写UDAF的4种姿势
date: 2019-05-11 20:15:05
tags: spark
categories:
- spark
---

摘要: 
> (探索解决sql的多行处理能力盲区)
1. 搭配collect_set+UDF;
2. RDD: combineByKey;
3. Dataframe: 继承UserDefinedAggregateFunction;
4. Dataset: 继承Aggregator。

   前文探索了解决sql对于单行处理的能力盲区(http://xiaoyue26.github.io/2019/05/08/2019-05/%E5%B0%86pyspark%E4%B8%AD%E7%9A%84UDF%E5%8A%A0%E9%80%9F50/ )，本文接着探索解决sql对于多行处理(UDAF/用户自定义聚合函数)的能力盲区。

# 姿势1：搭配collect_set+UDF
基本思想是强行把一个group行拼成一个数组，然后编写一个能处理数组的UDF即可。如果是多行变多行，则UDF的输出也构造成数组，然后用explode打开。如果想要把多行聚合成一行（类似于sum），则直接输出结果即可。
```python
def str_list2idfa(txt_list):
    try:
        res = list()
        for txt in txt_list:
            res.append(str2idfa(txt))
        return res
    except:
        return []


if __name__ == '__main__':
    spark = SparkSession.builder.appName(app_name).getOrCreate()
    provider = TDWSQLProvider(spark, user=user, passwd=passwd, db=db_name)

    in_df = provider.table(in_table_name, ['p_2019042100'])  # 分区数组
    print(in_df.columns)
    # 1. 创建udaf:
    str_list2idfa_udaf = udf(str_list2idfa  # 实际运行的函数
                            , ArrayType(ArrayType(StringType()))  # 函数返回值类型
                            )
    # 2. 在df中使用,将数组转成二维数组:
    out_t1 = in_df.groupBy('uin').agg(
        str_list2idfa_udaf(
            collect_set('value')
        ).alias('value_list')
    )
    print(out_t1.columns)
    out_t1.printSchema()
    out_t1.createOrReplaceTempView("t1")
    # 3. 将二维数组打开,一行变多行,一列变两列:
    out_t2 = spark.sql('''
    select uin
          ,idfa_idfv[0] as idfa
          ,idfa_idfv[1] as idfv
    from t1
    lateral view explode(value_list) tt as idfa_idfv
    ''')
    out_t2.printSchema()
    print(out_t2.take(10))
```
- 优点： 开发成本低，不用编译。
- 缺点： 性能一般，增加了转换数组、explode的成本，可能导致聚合过程完全在单点进行，对于数据倾斜的承受能力较低。

# 姿势2： 使用RDD的combineByKey算子
  上述方法本质上是用UDF强行模拟了UDAF的功能，性能上有所损失。第二种方法是使用RDD的`combineByKey`算子: 
```scala
    val spark = SparkSession.builder.appName("UdafDemo").getOrCreate()
    val rddProvider = new TDWProvider(spark.sparkContext, user, pass, db) // 这个返回rdd
    val inRdd = rddProvider.table("t_dw_dc0xxxx", Array("p_2019042100"))
    println("getNumPartitions:")
    println(inRdd.getNumPartitions)
    val kvRdd: RDD[(Long, String)] = inRdd
      .map(row => (row(3).toLong, UdfUtils.str2idfa(row(9))))
      .filter(x => x._2.isDefined)
      .map(x => (x._1, x._2.get))

    val combineRdd: RDD[(Long, mutable.Set[String])] = kvRdd
      .combineByKey(
        (v: String) => mutable.Set(v),
        (set: mutable.Set[String], v: String) => set += v,
        (set1: mutable.Set[String], set2: mutable.Set[String]) => set1 ++= set2)

    val outRdd: RDD[(Long, String)] = combineRdd.flatMap(kv => {
      val uin = kv._1
      val set = kv._2
      val res = mutable.MutableList.empty[(Long, String)]
      set.foreach(x => res += ((uin, x)))
      res.iterator
    })
    outRdd.take(10).foreach(println)
    // println(outRdd.count())
```
- 优点： 代码简洁，容易理解，性能高; 
- 缺点： 需要学习RDD相关知识。

# 姿势3: 使用Dataframe(继承UserDefinedAggregateFunction)
假设用户比较熟悉Dataframe操作，还可以通过继承`UserDefinedAggregateFunction`类编写一个完整的UDAF：
```scala
// part1: UDAF:
import org.apache.spark.sql.Row
import org.apache.spark.sql.expressions.{MutableAggregationBuffer, UserDefinedAggregateFunction}
import org.apache.spark.sql.types._

object DfUdaf extends UserDefinedAggregateFunction {
  def inputSchema: StructType = StructType(StructField("value", StringType) :: Nil)

  // Map[String,Null] 当Set用了
  def bufferSchema: StructType = new StructType().add("idfa_idfv", MapType(StringType, NullType))

  override def dataType: DataType = MapType(StringType, NullType)

  override def deterministic: Boolean = true

  override def initialize(buffer: MutableAggregationBuffer): Unit = {
    buffer.update(0, Map[String, Null]())
  }

  override def update(buffer: MutableAggregationBuffer, input: Row): Unit = {
    val map = buffer.getAs[Map[String, Null]](0)
    val value = input.getAs[String](0)
    val idfa_idfv = UdfUtils.str2idfa(value)
    if (idfa_idfv.isDefined) {
      buffer.update(0, map ++ Map(idfa_idfv.get -> null))
    }
  }

  override def merge(buffer1: MutableAggregationBuffer, buffer2: Row): Unit = {
    val map1 = buffer1.getAs[Map[String, Null]](0)
    val map2 = buffer2.getAs[Map[String, Null]](0)
    buffer1.update(0, map1 ++ map2)
  }

  override def evaluate(buffer: Row): Any = buffer.getAs[Map[String, Null]](0)
}
 
// part2: main函数:
    val spark = SparkSession.builder.appName("UdafDemo").getOrCreate()
    val sqlProvider = new TDWSQLProvider(spark, user, pass, db)

    // val rddProvider = TDWProvider(spark.sparkContext, user, pass, db) // 这个返回rdd
    val inDf = sqlProvider.table("t_dw_dc0xxxx", Array("p_2019042100"))
    println("getNumPartitions:")
    println(inDf.rdd.getNumPartitions)

    spark.udf.register("collect_idfa", DfUdaf)
    inDf.createOrReplaceTempView("t1")
    val outDf = spark.sql("" +
      "select uin,idfa_idfv " +
      "from " +
      "(select uin,collect_idfa(value) as vmap from t1 group by uin) a " +
      "lateral view explode(vmap) tt as idfa_idfv,n" +
      "")
    outDf.take(10).foreach(println)
```
优点: 可以直接在sql中引用，重用性高，性能高;
缺点: 开发成本高，只支持scala，需要编译。

# 姿势4： 使用Dataset（继承Aggregator）
如果用户对于Dataset的api比较熟悉，可以继承Aggregator开发UDAF:
```scala
// part1: UDAF:
import org.apache.spark.sql.{Encoder, Encoders}
import org.apache.spark.sql.expressions.Aggregator

class DsUdaf[IN](val f: IN => String) extends Aggregator[IN, Set[String], Set[String]] {

  override def zero: Set[String] = Set[String]()

  override def reduce(buf: Set[String], a: IN): Set[String] = {
    val idfa_idfv = UdfUtils.str2idfa(f(a))
    buf ++ idfa_idfv
  }

  override def merge(b1: Set[String], b2: Set[String]): Set[String] = b1 ++ b2

  override def finish(reduction: Set[String]): Set[String] = reduction

  override def bufferEncoder: Encoder[Set[String]] = Encoders.kryo[Set[String]]

  override def outputEncoder: Encoder[Set[String]] = Encoders.kryo[Set[String]]

}
// part2: main函数:
val spark = SparkSession.builder.appName("UdfDemo").getOrCreate()
    val sqlProvider = new TDWSQLProvider(spark, user, pass, db)

    // val rddProvider = TDWProvider(spark.sparkContext, user, pass, db) // 这个返回rdd
    val inDf = sqlProvider.table("t_dw_dc0xxxx", Array("p_2019042100"))
    println("getNumPartitions:")
    println(inDf.rdd.getNumPartitions)

    import spark.implicits._
    inDf.createOrReplaceTempView("t1")
    val df2 = spark.sql("select uin,value from t1")
    df2.printSchema()

    val inDS = df2.as[UinValue]
    // inDS.take(10).foreach(println)
    val outDs: Dataset[(Long, Set[String])] = inDS.groupByKey(_.uin).agg(new DsUdaf[UinValue](_.value).toColumn)
    // outDs.take(10).foreach(println)
    val ds2 = outDs.flatMap(pair => {
      val uin = pair._1
      val idfa_set = pair._2
      idfa_set.map(idfa => (uin, idfa))
    })
    ds2.printSchema()
    ds2.take(10).foreach(println)
```
其中`Encoder`部分由于还不支持Set集合类型，可以使用kryo序列化成二进制。（更多Encoder相关参见:http://xiaoyue26.github.io/2019/04/27/2019-04/spark%E4%B8%AD%E7%9A%84encoder/ ）

优点: 类型安全，继承Aggregator开发的成本略小于继承UserDefinedAggregateFunction;
缺点: 只支持scala，需要编译。

# 总结
本文总结了在Rdd,Dataframe,Dataset三种api下编写UDAF的方法（三种api的对比参见http://xiaoyue26.github.io/2019/04/29/2019-04/spark%E4%B8%ADRDD%EF%BC%8CDataframe%EF%BC%8CDataSet%E5%8C%BA%E5%88%AB%E5%AF%B9%E6%AF%94/ ），以及使用UDF模拟UDAF功能的方法。大家可以根据自己熟悉的api和需求选择。
- 如果不在意性能：用`collect_set`+`UDF`模拟一个；(姿势1)
- 如果在意性能，但是只用一次: 可以直接用RDD的`combineByKey`，代码较短；（姿势2） 
- 如果在意性能，而且会反复复用: 建议使用Dataframe，继承`UserDefinedAggregateFunction`编写一个UDAF。（姿势3）

