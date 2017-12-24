---
title: hive udf
date: 2017-01-15 19:29:01
tags: 
- java 
- hive
categories:
- hadoop
- hive
---

中位数算法:
https://segmentfault.com/a/1190000008322873
http://www.voidcn.com/article/p-yzrnuwwi-bdv.html

# hive类型和java类型
<table>
<tr><td>Hive column type</td>	<td>UDF types</td></tr>
<tr><td>string	</td>	
<td>java.lang.String, org.apache.hadoop.io.Text</td></tr>
<tr><td>int	</td>	
<td>int, java.lang.Integer, org.apache.hadoop.io.IntWritable</td></tr>
<tr><td>boolean	</td>	
<td>bool, java.lang.Boolean, org.apache.hadoop.io.BooleanWritable</td></tr>
<tr><td>array<type>	</td>	<td>java.util.List<Java type></td></tr>
<tr><td>map<ktype, vtype>	</td>	
<td>java.util.Map<Java type for K, Java type for V></td></tr>
<tr><td>struct	</td>	<td>Don't use Simple UDF, use GenericUDF</td></tr>
</table>


# UDAF
http://www.cnblogs.com/ggjucheng/archive/2013/02/01/2888051.html
`GenericUDAFEvaluator`的几种`MODE`定义如下:
```
public static enum Mode {
    /**
     * PARTIAL1: 这个是mapreduce的map阶段:从原始数据到部分数据聚合
     * 将会调用iterate()和terminatePartial()
     */
    PARTIAL1,
        /**
     * PARTIAL2: 这个是mapreduce的map端的Combiner阶段，负责在map端合并map的数据::从部分数据聚合到部分数据聚合:
     * 将会调用merge() 和 terminatePartial() 
     */
    PARTIAL2,
        /**
     * FINAL: mapreduce的reduce阶段:从部分数据的聚合到完全聚合 
     * 将会调用merge()和terminate()
     */
    FINAL,
        /**
     * COMPLETE: 如果出现了这个阶段，表示mapreduce只有map，没有reduce，所以map端就直接出结果了:从原始数据直接到完全聚合
      * 将会调用 iterate()和terminate()
     */
    COMPLETE
  };
```

特别提醒：
Merge函数的输入参数是(State other),因为other对象会被复用,因此other里的成员的使用不能是浅拷贝,(比如直接塞到当前state里),可以用深拷贝.(对于List,Map等容器)



系统内置的UDAF函数可以从:`FunctionRegistry`类中查看.
流程控制更为精细的UDAF,AVG:
`org.apache.hadoop.hive.ql.udf.generic.GenericUDAFAverage`

hive调用java方法:
select java_method("java.lang.Math","max",1,2);


# UDF数据类型相关
```
select array(1,2,3) -- => [1,2,3]
select array(1,'2',3) -- => ['1','2','3'] 只要有一个string,就全是string
select array(1,NULL,3) -- => [1,NULL,3] NULL则不影响类型转换. 
```
