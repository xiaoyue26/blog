---
title: 'mysql,sqoop,hive分隔符总结'
date: 2017-10-01 19:30:24
tags: 
- mysql 
- sqoop 
- hive
categories:
- hadoop
- hive
---


# mysql分隔符
>Mysql默认的分隔符设置，字段之间以逗号`,`隔开，行之间以换行`\n`隔开，默认转义符号是`\`，字段值以单引号`'`包含起来。

# sqoop分隔符
> sqoop可以设定分隔符按照mysql的约定.
使用配置 `--mysql-delimiters`.
也可以自定义以下设置:

## `--enclosed-by <char>`  (所有字段)
>给字段值前后加上指定的字符，比如双引号，示例：--enclosed-by '\"'，显示例子："3","jimsss","dd@dd.com"

## `--optionally-enclosed-by <char>` (含有单(双)引号的字段)
一般指定这一项为"'",而不指定`--enclosed-by <char>` 

## `--escaped-by <char>`
转义时使用的char,一般为"\"

## `--fields-terminated-by <char>`
字段分隔字段,默认为`,`. 

## `--lines-terminated-by <char>`
行分隔符,默认为'\n'

# hive分隔符
> hive不可以设置行分隔符,hive的行分隔符只能是'\n'.
hive可以设置的分隔符如下:

```sql
ROW FORMAT DELIMITED
  FIELDS TERMINATED BY '\t'
  COLLECTION ITEMS TERMINATED BY ','
  MAP KEYS TERMINATED BY ':'
```
这些也可以设置为`\0001`,`\0002`,`\0003`.

由于hive不能自定义行分隔符,因此sqoop导入hive时只能指定行分隔符为`\n`,因此sqoop导入hive的时候需要过滤掉hive默认使用的分隔符,可以使用参数:
```sql
--hive-drop-import-delims
```
作用是把字符串类型的字段中的\r,\n,\001等字符丢弃. 
这个参数可以与`--mysql-delimiters`同时使用.
也可以使用
```sql
--hive-delims-replacement
` `
```
将对应字符换成空格,以保持原字符串的结构. 

# Solution
1)
由于hive的csv解析bug太多,使用sqoop和hive组合时,应该使用hive的分隔符,sqoop中配置`\0001`和hive中配置'\u0001'. 
1.1)hive对于\'和'的识别是混乱的,如下述的构造数据,解析起来是有问题的:
```
\'异常的第一列,第二列
```
这行数据解析出来每一列都是NULL.

1.2)另一种解析错误的数据如下:
```
\'异常的第一列,'第二列,'
或:
\\'异常的第一列,'第二列,'
```
这两行解析的结果均为:
```
//第一列为:
异常的第一列,第二列	
//第二列为:
NULL
```
而实际我们期望的解析结果是:
```
//第一列为:
'异常的第一列
//第二列为:
第二列,
```
hive这里解析的逻辑是,把
```
\'异常的第一列,'
```
看做一个整体,继续拼接剩余的字符串,直到遇到逗号.这种解析方法是错误的.
 
2)
一种只对这种情况有效的解决方案,使用参数`--enclosed-by <char>`
```sql
--enclosed-by
"'"
```
使用这种方法以后的数据会变成:
```
'\'异常的第一列','第二列'
```
此时可以用`org.apache.hadoop.hive.serde2.OpenCSVSerde`类正常解析,注意这里若使用`--optionally-enclosed-by <char>`参数,
并不能自动加上第一列的单引号.
2.1)sqoop仅在它认为字段中出现了逗号时才自动加上单引号;
2.2)sqoop对于字段中出现的逗号进行转义,变为\';
2.3)对于hive而言,出现单引号时,它会把这些单引号对内部的部分认为是一个整体,对这个整体内的逗号进行忽略. 但hive的bug之处在于,它遇到\'时候,也会试图去匹配下一个\'或'.

3) 使用csv的opt:
```sql
import
-D
mapreduce.map.speculative=false
--append
--connect
jdbc:mysql://apolo-publisher-reader:3306/apolo?useUnicode=true&characterEncoding=UTF-8
--username
xxx
--password
xxx@xxx
--boundary-query
'select unix_timestamp("2017-07-23")*1000,unix_timestamp(date_add("2017-07-23",interval 1 day))*1000'
--query
SELECT  * FROM question where updatedTime between unix_timestamp("2017-07-23")*1000 and unix_timestamp(date_sub("2017-07-23",interval -1 day))*1000 and $CONDITIONS
--split-by
updatedTime
--hive-delims-replacement
' '
--target-dir
/user/fengmq01/test5/
--enclosed-by
"'"
--m
20
--direct
```


4)最安全的solution,不使用csv:
```sql
import
-D
mapreduce.map.speculative=false
--append
--connect
jdbc:mysql://apolo-publisher-reader:3306/apolo?useUnicode=true&characterEncoding=UTF-8
--username
xxx
--password
xxx@xxx
--boundary-query
'select unix_timestamp("2017-07-23")*1000,unix_timestamp(date_add("2017-07-23",interval 1 day))*1000'
--query
SELECT  * FROM question where updatedTime between unix_timestamp("2017-07-23")*1000 and unix_timestamp(date_sub("2017-07-23",interval -1 day))*1000 and $CONDITIONS
--split-by
updatedTime
--hive-delims-replacement
' '
--target-dir
/user/fengmq01/test5/
--fields-terminated-by
"\0001"
--m
20
--direct
```

对应的hive外部表参数:
```sql
ROW FORMAT DELIMITED
FIELDS TERMINATED BY '\u0001'
COLLECTION ITEMS TERMINATED BY '\u0002'
MAP KEYS TERMINATED BY '\u0003'
```


### csv外部表语句:
```sql
drop table temp.feng_test1;
create table temp.feng_test1
(c1 string
,c2 string
,c3 string
,c4 string
)
;
load data local inpath 'input.csv' into table temp.feng_test1;

create external table temp.feng_test2
(c1 string
,c2 string
,c3 string
,c4 string
)
ROW FORMAT SERDE
  'org.apache.hadoop.hive.serde2.OpenCSVSerde'
WITH SERDEPROPERTIES (
  'escapeChar'='\\',
  'quoteChar'='\'',
  'separatorChar'=',')
STORED AS INPUTFORMAT
  'org.apache.hadoop.mapred.TextInputFormat'
OUTPUTFORMAT
  'org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat'
LOCATION
  'hdfs://f04/user/hive/warehouse/temp.db/feng_test1'
```
