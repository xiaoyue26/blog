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
// 将数字(JVM对象)转换为DataSet中的元素
// 这里由于是常见的原始类型，所以spark提供了隐式encoder的调用，隐藏了这些细节。
```
Encoder将jvm转换为堆外内存二进制，使用成员位置信息，降低反序列化的范围（反序列化需要的列即可）。
// (类似于Hive中的反序列化,把kv转换为row)

Encoder不要求线程安全。

# Why: 为啥用Encoder?
stackoverflow上说encoder消耗更少的内存。因为kryo把`dataSet`中的所有行都变成了一个打平的二进制对象。
`10x faster than Kryo serialization (Java serialization orders of magnitude slower)`
DataFrame本质上是DataSet[Row]，用的固定是RowEncoder，所以不需要传Encoder。
Encoder底层是钨丝计划的堆外内存优化，节省了jvm对象头、反序列化、gc的开销。

# When: 啥时候使用Encoder
`Encoder`适用于原始类型、case class对象(因为有默认的apply/unapply方法)、spark-sql类型。

Encoder支持的类型非常多，不支持的情况：
```
1. 如果类型是javabean，类的成员如果是容器，只能是List，不能是其他容器（还没有实现）;
2. 不支持大于5的Tuple；
3. 不支持`Option`；
4. 不支持`null`值的`case class`。
```
不支持的时候，可以把不支持的部分用kyro-Encoder，相当于不支持的部分直接当做一个二进制，不享受优化，但其他不支持部分可以享受优化。

# How: 怎么使用Encoder:
显式： 使用`Encoders`类(类似于utils)的静态工厂方法;
隐式：`import spark.implicits._`: 原始类型和`Product`类型(也就是`case class`)可以直接隐式支持。
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



## 源码阅读
### Encoder
```
trait Encoder[T] extends Serializable {
  /** Returns the schema of encoding this type of object as a Row. */
  def schema: StructType
  /**
   * A ClassTag that can be used to construct and Array to contain a collection of `T`.*/
  def clsTag: ClassTag[T]
  // 存了ClassTag的话，就能在运行时构建泛型的数组了。
}
```
`TypeTag`: 相当于scala以前的`Manifest`,用于存储泛型参数的实际类型。(泛型参数的实际类型运行时会被JVM擦除，有了`TypeTag`就能在运行时获得实际类型了)
`ClassTag`: 相当于scala以前的`ClassManifest`,功能大致同上，但存得少些，比如如果是泛型的泛型，参数是泛型数组`List[T]`,`TypeTag`能全部存下，`ClassTag`就存一个`List`。
```
classTag[List[Int]]
//scala.reflect.ClassTag[List[Int]] =↩
//        ClassTag[class scala.collection.immutable.List]
typeTag[List[Int]]
//
// reflect.runtime.universe.TypeTag[List[Int]] = TypeTag[scala.List[Int]]
```


## ExpressionEncoder
Encoder的内置唯一实现类。
`jvm对象<=>内部行格式`: 钨丝计划unsafeRow、expressions提取case class的变量名。
可以支持Tupple但不支持`Option`和`和null`值的`case class`。

它会生成变量名name和位置的绑定，以便钨丝计划的`code gen`使用`unsafe row`.
Tupple最多到5.

`Serializer`: raw object=>InternalRow, 用expression解析提取对象值；
`Deserializer`: InternalRow=>raw object,用expression构造对象。 
因为unsafeRow是二进制存放在堆外，所以转换成row看做序列化。

### Encoders
(注意比Encoder多一个s)
提供了很多静态工厂方法获得Encoder(实际上目前获得的都是`ExpressionEncoder`)
大致可以分为几类:
1. java原始类型:  `Encoders.BOOLEAN`等
2. scala原始类型: `Encoders.scalaBoolean`等.(多一个scala前缀)
3. javaBean类型: `bean[T](beanClass: Class[T])`。但目前成员只支持List容器，不支持其他的容器。支持原始类型或嵌套javaBean。
4. kryo序列化类型: `kryo[T: ClassTag]`；
5. java序列化类型: `javaSerialization[T: ClassTag]`；
6. Tuple类型: 从Tuple2到Tuple5.
7. Product类型: 也就是`case class`.

其中前三种是直接调用`ExpressionEncoder`，第四第五种本质上是间接调用了`ExpressionEncoder`:
```
ExpressionEncoder[T](
      schema = new StructType().add("value", BinaryType),
      flat = true,
      serializer = Seq(
        EncodeUsingSerializer(
          BoundReference(0, ObjectType(classOf[AnyRef]), nullable = true), kryo = useKryo)),
      deserializer =
        DecodeUsingSerializer[T](
          Cast(GetColumnByOrdinal(0, BinaryType), BinaryType),
          classTag[T],
          kryo = useKryo),
      clsTag = classTag[T]
    )
```
因此第四第五后两种序列化本质上是把整个对象看做一个二进制类型，不利于后续优化和减少反序列化。

原始类型还包括:
```
java的:
byte,short,int,long,float,double,java.math.BigDecimal,java.sql.Date,java.sql.Timestamp
scala的:
Array[Byte],byte,short,int,long,float,double
```