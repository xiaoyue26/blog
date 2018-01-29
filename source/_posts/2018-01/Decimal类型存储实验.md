---
title: Decimal类型存储实验
date: 2018-01-28 10:45:02
tags: mysql
categories:
- mysql

---


- 为啥要实验:
<高性能mysql>第四章中提到`decimal`类型的存储空间时,用`decimal(18,9)`举例,说前9个数字占4B,后9个数字占4B,小数点占1B,总共9B. 
我觉得小数点也要占1B实在诡异,查了mysql官方文档和其他网上资料也没看到说小数点也占一个字节的说法,于是实验一番.

## 步骤1: 准备数据
- 建表语句:
```
CREATE TABLE `du_decimal` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `dump_data` varchar(45) DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=MyISAM AUTO_INCREMENT=34 DEFAULT CHARSET=utf8
```

由于`Innodb`把数据和索引都存在了一个文件`.idb`里,为了只关注数据的存储空间占用,这里使用`MyIsam`引擎,只关注`.MYD`文件即可.

```
插入数据17行后容量:
$ ll du_decimal.MYD
-rw-rw---- 1 maintain maintain 340 Jan 29 09:10 du_decimal.MYD
```
之所以是17行,主要是想选个质数.XD

# 步骤2: 插入Decimal列
下表为新增不同Decimal列后数据文件的大小:
| Decimal列     | `.MYD`大小(B)   | 列新增大小(B)(每行)
| :--------:    | :-----:| :-----:|
| 无            | 340     | 0 |
| Decimal(1,0)  | 408    | 4 |
| Decimal(2,0)  | 408   |4 |
| Decimal(2,1)  | 408   |4 |
| Decimal(3,0)  | 408   |4 |
| Decimal(4,0)  | 408   |4 |
| Decimal(4,1)  | 408   |4 |
| Decimal(5,0)  | 408   |4 |
| Decimal(6,0)  | 408   |4 |
| Decimal(7,0)  | 408   |4 |
| Decimal(8,0)  | 408   |4 |
| Decimal(9,0)  | 408   |4 |
| Decimal(18,9) | 476   |8 |
| Decimal(20,6) | 544   |12 |

由此可以看出Decimal(18,9)占了8B,而不是书上说的9B.(小数点不需要1B)
但因为操作系统最小分配4B,所以9位数字以内的存储占用都是4B. 

## Tips
此外,如果没有登录数据库那台机器的权限,可以使用sql进行数据文件大小查询:
```
SELECT TABLE_SCHEMA as db
      ,table_name
      ,data_length
FROM information_schema.TABLES
where  TABLE_NAME='du_decimal'
```
