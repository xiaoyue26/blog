---
title: Hive Serde
date: 2017-01-14 19:26:56
tags: hive
categories:
- hadoop
- hive
---

# Serde相关
https://www.coder4.com/archives/4031
反序列化原理(`Deserialize`):
```
HDFS files –> InputFileFormat –> <key, value> –> Deserializer –> Row object
```
序列化原理(`Serialize`):
```
Row object –> Serializer –> <key, value> –> OutputFileFormat –> HDFS files
```
>hive的serde内置:
Avro
ORC
RegEx
Thrift
Parquet
CSV
JsonSerDe

### 自定义Serde:
>需要extends AbstractSerDe, 实现6个方法:
initialize
,deserialize,getSerializedClass,serialize
,getSerDeStats,getObjectInspector
参见Hive1.2.1的openCSV实现,可以总结各个方法的作用.

1.`initialize`方法:
```
读取hadoop配置和表配置(建表语句中的),准备一些和元数据有关的数据.
比如所有列的类型,输出列的size,分隔符等等.将这些信息存到对象里备用.
```

2.`serialize`方法:
```
序列化. 
由原理可知,输入是一个Row对象,输出是一些Key,value对,以便被OutputFileFormat解析.
由于OpenCSV使用的OutputFileFormat是
org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat
因此,这个方法的输出要与之吻合.
    /**
     * 输入: 行对象 (行数据Obj,行类型ObjOI)
     * 输出: Key,value.  要和hive默认的outputFileFormat吻合.
     * 即要和 org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat
     * 的输入相同. 而这个类是忽略Key的,因此这个方法的实际输出只有Value.
     * */
OpenCSV里的大致内容就是把行类型ObjOI转成StructOI,
然后把每一列的类型取出来,再用类型把数据取出来,再把数据全转成String,
传给CSVWriter,写到一个StringWriter里,然后转成一个Text返回.
```

3.`deserialize`方法:
```
反序列化.
由原理可知,输入是一个Key,Value,输出是一个行对象.
其中Key,Value由InputFileFormat生成,对于OpenCSV来说就是默认的Hive设定:
org.apache.hadoop.mapred.TextInputFormat
输出实际上是一个List<String>. 

OpenCSV里的大致内容就是把输入转成Text再转成String然后用CSVReader读出来,
写到一个String[]里,再存到一个List<String>里(补一些缺的,确保列数一样),然后返回.
```

4.`getObjectInspector`方法:
```
返回整个表的结构类型OI.
从内容上看返回了StructOI.(包括列名和列数,每一列的类型)
对于OpenCSV来说每一列都是String.
```

5.`getSerializedClass`方法:
```
获取写的时候返回的类.
对于OpenCSV来说,返回Text.class.
因为它的Serialize方法的下游是hive的
org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat
而且Serialize的实现实际上也返回了一个Text.(声明的只是Writable)
```

6.`getSerDeStats`方法:
```
返回序列化类的写入状态信息,具体包括RawDataSize和RowCount. 也就是原始数据大小和行数.
Hive1.2.1实现的不是很健全.自己用得也很少.
只在两处提取了这个信息:
1. FileSinkOperator. 写入数据的时候获取了一下RawDataSize.
2. MapOperator. 读取数据的时候获取了一下RawDataSize.

甚至对于openCSV,它直接返回了一个null.
对于Hive1.2.1,我们自定义实现Serde的时候,也可以返回null,因为Hive1.2.1的源码中对于这个返回值都是有null判定的. 但如果要对性能做进一步研究,要记录这两个数据,可以参考
LazySimpleSerDe中的实现.

```
