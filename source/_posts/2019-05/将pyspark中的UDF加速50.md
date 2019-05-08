---
title: 将pyspark中的UDF加速50%
date: 2019-05-08 22:39:03
tags: spark
categories:
- spark

---

# 摘要
> 调用jar中的UDF，减少python与JVM的交互，简单banchmark下对于50G数据集纯map处理可以减少一半处理时间。
牺牲UDF部分的开发时间，尽量提高性能。
以接近纯python的开发成本，获得逼近纯scala的性能。兼顾性能和开发效率。

# 背景
  当遇到sql无法直接处理的数据时(比如加密解密、thrift解析操作二进制)，我们需要自定义函数(UDF)来进行处理。出于开发效率的考虑，我们一般会选择tesla平台，使用pyspark脚本。
  
# Before: 最简单的UDF
一个最简单的UDF处理大致如下:
```python
def str2idfa(txt):
    try:
        txtDecoded = base64.b64decode(txt)
        bytesWithSalt = bytes(txtDecoded)
        # 省略实际处理代码
        return 'dump_data'
    except:
        print('error here')
        return '-1#-1'


if __name__ == '__main__':
    spark = SparkSession.builder.appName(app_name).getOrCreate()
    in_provider = TDWSQLProvider(spark, user=user, passwd=passwd, db=db_name)

    in_df = in_provider.table('t_dw_dc0xxxx', ['p_2019042100'])  # 分区数组
    print(in_df.columns)
    in_df.createOrReplaceTempView("t1")
    # 1. 注册udf:
    spark.udf.register("str2idfa", str2idfa, StringType())
    # 2. 在sql中使用:
    out_t1 = spark.sql('''select uin
        ,str2idfa(value) as idfa_idfv
        from t1
        ''')
    print(out_t1.columns)
    print(out_t1.take(10))
```

## 底层实现原理
{% img /images/2019-05/pyspark_call.png 800 1200 pyspark_call %}
  如上图所示，pyspark并没有像dpark一样用python重新实现一个计算引擎，依旧是复用了scala的jvm计算底层，只是用py4j架设了一条python进程和jvm互相调用的桥梁。
  `driver`:  pyspark脚本和sparkContext的jvm使用py4j相互调用;
  `executor`: 由于driver帮忙把spark算子封装好了，执行计划也生成了字节码，一般情况下不需要python进程参与，仅当需要运行UDF(含lambda表达式形式)时，将它委托给python进程处理(DAG图中的`BatchEvalPython`步骤)，此时JVM和python进程使用socket通信。
  
   上述使用简单UDF时的pyspark由于需要使用UDF，因此DAG图中有`BatchEvalPython`步骤:
   {% img /images/2019-05/py_udf.png 400 400 py_udf %}
  
## BatchEvalPython过程
参考源码：https://github.com/apache/spark/blob/master/sql/core/src/main/scala/org/apache/spark/sql/execution/python/BatchEvalPythonExec.scala
可以看到和这个名字一样直白，它就是每次取100条数据让python进程帮忙处理一下:
```scala
// 第58行:
// Input iterator to Python: input rows are grouped so we send them in batches to Python.
    // For each row, add it to the queue.
    val inputIterator = iter.map { row =>
      if (needConversion) {
        EvaluatePython.toJava(row, schema)
      } else {
        // fast path for these types that does not need conversion in Python
        val fields = new Array[Any](row.numFields)
        var i = 0
        while (i < row.numFields) {
          val dt = dataTypes(i)
          fields(i) = EvaluatePython.toJava(row.get(i, dt), dt)
          i += 1
        }
        fields
      }
    }.grouped(100).map(x => pickle.dumps(x.toArray))
```
    

   由于我们的计算任务一般耗时瓶颈在于executor端的计算而不是driver，因此应该考虑尽量减少executor端调用python代码的次数从而优化性能。
   
参考源码：https://github.com/apache/spark/blob/master/python/pyspark/java_gateway.py
```python
// 大概135行的地方:
# Import the classes used by PySpark
java_import(gateway.jvm, "org.apache.spark.SparkConf")
java_import(gateway.jvm, "org.apache.spark.api.java.*")
java_import(gateway.jvm, "org.apache.spark.api.python.*")
java_import(gateway.jvm, "org.apache.spark.ml.python.*")
java_import(gateway.jvm, "org.apache.spark.mllib.api.python.*")
# TODO(davies): move into sql
java_import(gateway.jvm, "org.apache.spark.sql.*")
java_import(gateway.jvm, "org.apache.spark.sql.api.python.*")
java_import(gateway.jvm, "org.apache.spark.sql.hive.*")
java_import(gateway.jvm, "scala.Tuple2")
```
pyspark可以把很多常见的运算封装到JVM中,但是显然不包括我们的UDF。
所以一个很自然的思路就是把我们的UDF也封到JVM中。

# After: 调用JAR包中UDF
首先我们需要用scala重写一下UDF:
```scala
object UdfUtils extends java.io.Serializable {

  case class Idfa(idfa: String, idfv: String) {
    private def coalesce(V: String, defV: String) =
      if (V == null) defV else V

    override def toString: String = coalesce(idfa, "-1") + "#" + coalesce(idfv, "-1")
  }

  def str2idfa(txt: String): Option[String] = {
    try {
      val decodeTxt: Array[Byte] = Base64.getDecoder.decode(txt)
      // TODO 省略一些处理逻辑
      val str = "after_some_time"
      val gson = new Gson()
      val reader = new JsonReader(new StringReader(str))
      reader.setLenient(true)
      val idfaType: Type = new TypeToken[Idfa]() {}.getType
      Some(gson.fromJson(reader, idfaType).toString)
    }
    catch {
      case e: Throwable =>
        println(txt)
        e.printStackTrace()
        None
    }
  }
  // 关键是这里把普通函数转成UDF:
  def str2idfaUDF: UserDefinedFunction = udf(str2idfa _)

```

然后在pyspark脚本里调用jar包中的UDF:
```python
#!/usr/bin/env python
# -*- coding: utf-8 -*-
import sys
from pytoolkit import TDWSQLProvider, TDWUtil, TDWProvider
from pyspark import SparkContext, SQLContext
from pyspark.sql import SparkSession, Row
from pyspark.sql.types import StructType, LongType, StringType, StructField, IntegerType
from pyspark.sql.functions import udf, struct, array
from pyspark.sql.column import Column
from pyspark.sql.column import _to_java_column
from pyspark.sql.column import _to_seq
from pyspark.sql.functions import col

def str2idfa(col):
    _str2idfa = sc._jvm.com.tencent.kandian.utils.UdfUtils.str2idfaUDF()
    return Column(_str2idfa.apply(_to_seq(sc, [col], _to_java_column)))


spark = SparkSession.builder.appName(app_name).getOrCreate()
sc = spark.sparkContext
if __name__ == '__main__':
    in_provider = TDWSQLProvider(spark, user=user, passwd=passwd, db=db_name)
    in_df = in_provider.table('t_dw_dcxxxx', ['p_2019042100'])  # 分区数组
    print(in_df.columns)
    in_df.createOrReplaceTempView("t1")
    out_t1 = in_df.select(col('uin')
                          , str2idfa(col("value"))) # 直接使用scala的udf,节省43%时间,减少两个transform
    print(out_t1.columns)
    print(out_t1.take(10))
```
其中`_jvm`变量是`sparkContext`中`JVMView`对象的名字,此外sc中还有`_gateway`变量以连接JVM中的`GatawayServer`。
提交时，在tesla上的配置`spark-conf`jar包路径:
```
spark.driver.extraClassPath=pipe-udf-1.0-SNAPSHOT-jar-with-dependencies.jar
spark.executor.extraClassPath=pipe-udf-1.0-SNAPSHOT-jar-with-dependencies.jar
```
同时在依赖包文件中上传jar包。

这样一通操作之后，DAG图变成了这样:
{% img /images/2019-05/py_jar.png 400 400 py_jar %}

可以看到比之前少了两个transform,没有了`BatchEvalPython`，也少了一个`WholeStageCodeGen`。
经过简单banchmark，对于50G数据集纯map处理。
第一种方案：大约13分钟；
第二种方案：大约7分钟。
第二种方案大约能节省一半的时间，并且进一步测试使用scala完全重写整个计算，运行时间和第二种方案接近，也大约需要7分钟。
# 总结
   在pyspark中尽量使用spark算子和spark-sql，同时尽量将UDF(含lambda表达式形式)封装到一个地方减少JVM和python脚本的交互。
   由于`BatchEvalPython`过程每次处理100行，也可以把多行聚合成一行减少交互次数。
   最后还可以把UDF部分用scala重写打包成jar包，其他部分则保持python脚本以获得不用编译随时修改的灵活性，以兼顾性能和开发效率。
