---
title: hive之bug汇总
date: 2017-12-25 18:01:19
tags: hive
categories:
- hadoop
- hive

---

这里总结一下hive的bug,或者说表现与spark sql不同的feature(bug?).
由于hive的distinct实际实现为group by,因此下述的group by相关bug也适用于distinct.


# 1. 列重命名BUG
> 导致结果错误.
spark-sql能正常处理.

子查询中重命名列时,如果和原有表中某列名相同,并且where条件中有那一列,取原有表的列值.
构造测试用例:
```
SELECT * FROM
(SELECT 123 as paperid
FROM  temp.feng_test1
where paperid=70455
)AS a
```
上述查询的结果是70455,而不是我们想象中的123. 
而这个查询:
```
SELECT * FROM
(SELECT 123 as paperid
FROM  temp.feng_test1
)AS a
```
或这个查询:
```
SELECT * FROM
(SELECT 123 as paperid
FROM  (select 70455 as paperid) as t 
where paperid=70455
)AS a
```
都能正确返回123.

# 2. GROUP BY+UDF+Serde复合bug
> 导致抛异常退出.
spark-sql能正常处理.

GROUP BY,自定义UDF和自定义Serde都能正常独立工作.
这是一个多重条件下产生的bug:
1.表定义为string,而自定义的serde类放入了long对象;(bug)
2.自定义UDF使用简便写法(继承UDF,复杂写法为继承GenericUDF);(正常行为)
3.运行如下语句:
```
select xx
from `1中serde的表`
group by `2中udf`
```

这个bug可以说是serde写得有问题导致的,严格来说hive没有太大问题.
原因是hive调用简单udf时,是运行时进行反射,填入方法的参数,实参与形参定义不同抛出异常.

# 3. GROUP BY+重复列 BUG
> 导致结果错误.
spark-sql能正常处理.

GROUP BY 时,如果有涉及引用的重复列, 如构造用例中的alist[0],由于hive在各种方面都会重用引用,会导致bug.
用例的输出结果为: `1,1,1`.
而不是我们想象中的: `1,1,3333`.

构造用例如下:
```
SELECT aid,bid,mistake
FROM
      (SELECT 1 as aid
      )AS a
JOIN
      (SELECT alist[0] as bid
            ,mistake
      FROM (SELECT 2 as courseid) AS a
      JOIN (SELECT 2 as courseid
                  ,'3333' as mistake
                  ,array(1) as alist
            ) AS b
        ON a.courseid=b.courseid
      GROUP BY alist[0]
              ,alist[0]
              ,mistake
      )AS b
 ON aid=bid
```

# 4. row_number + 数据类型溢出 bug
> 导致结果错误.
spark-sql能正常处理.

假设tp是一个字符串类型,强行转换成int类型时,如果发生数据溢出,比如值是13位时间戳(1514285700375),排序行为将不可预测.既不是降序也不是升序.

构造错误样例如下:
```
select userid,phaseid,tp
,row_number()over(partition by userid order by int(tp)) as rank
from xxxx
```

正确样例:
```
select userid,phaseid,tp
,row_number()over(partition by userid order by bigint(tp)) as rank
from xxxx
```

# 5. sort_array BUG(Feature) 函数副作用
> 导致结果错误.
spark-sql能正确处理.

hive中对数组进行排序后,会改变原有数组.(会在原有数组基础上排序)
spark-sql则会返回一个深拷贝,不改变原有数组.

```
select alist
      ,sort_array(alist) as alist2
FROM
(select array(1,3,2) as alist
)AS t
```

