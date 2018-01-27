---
title: 高性能Mysql笔记-第4章
date: 2018-01-27 22:18:58
tags: mysql
categories:
- mysql
---

# 第四章 Schema与数据类型优化
## 4.1 数据类型
原则:
>1. 更小的通常更好,  
int 好于 char (如可以用int存储ip);
date/time/datetime 好于 char. 
2. 尽量`Not Null`; // 因为NULL导致索引需要额外字节.// 不过这个影响不大

## 4.1.1 整数
包括:
```
TinyInt : 8位 
SmallInt: 16位
MediumInt: 24位
Int: 32位
BigInt: 64位
```
如果加上`unsigned`,正数范围提高一倍.
对于计算来说,一般都会转化成BigInt进行计算;
聚合函数则转化成`Decimal`或`Double`.

`Int(11)`中的11没有意义,只是对于GUI或控制台来说显示长度为11. 

## 4.1.2 实数
包括:
```
Decimal: 类似于字符串. 精确小数,使用mysql实现的计算;
Float : 4字节. 不精确,更快,使用cpu原生计算;
Double: 8字节. 不精确,更快.
```

Decimal(18,9): 
总长18,小数点右边9个数字.(所以左边也是9个数字)
左边9个数字占4个字节,右边9个数字占4个字节.
总共8个字节.

- Decimal的存储空间计算:
mysql把每9位数字存成4个字节. (转化成2进制字符串存储)
如果不足9位数字,存储空间消耗映射如下:
>1-2位=> 1字节
 3-4位=> 2字节
 5-6位=> 3字节
 7-9位=> 4字节
因此对于Decimal(20,6),左边14位数字需要空间=4+3=7B;右边6位需要3B,因此总共需要10B.


Decimal最多65个数字. 如果是Decimal(P,D),则P<=65,D<=30,且D<=P.
FLOAD,Double计算时转化为Double.

如果想要精确计算又想速度快(如财务数据):
> 使用BigInt,如果精度是1/10000,把所有数据乘以10000即可.



---- 
前面两章的内容:
# 第二章 Benchmark
# 第三章 服务器性能剖析
查看某条查询执行的时间分布:
```
set profiling=1;
-- select * from xxx ;
show profiles;
show profile for query 1;
```
查看某查询的执行计划:
```
explain select xxx;
```

sending data中的行为包括:
```
关联时搜索匹配的行
...
```