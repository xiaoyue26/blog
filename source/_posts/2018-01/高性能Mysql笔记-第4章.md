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

ip转换函数: Inet_AtoN(), Inet_NtoA(). 

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

## 4.1.3 字符串
字符串需要关注的方面:
1. 字符集
2. 排序规则(校对规则/collation)
3. 存储方式: 与存储引擎的具体实现有关.

字符串类型包括:
1. char: 定长. 填充使用空格. 会截断末尾空格.
2. varchar: 变长. (但如果使用ROW_FORMAT=FIXED,则会变成定长, 短的会填充补齐)
3. Binary: 存储二进制字符串. (字节码) 填充时使用'\0'而不是空格.
4. VarBinary: 变长二进制. 
5. BloB: 很长的二进制. 
6. Text: 很长的字符串.

**`varchar`的存储**
长度<=255: 1个字节存储长度信息,其余存储数据;
长度>255:  2个字节存储长度信息,其余存储数据.

latin1字符集: 
varchar(10)存储空间= 1+10=11B.
varchar(1000)存储空间= 2+1000=1002B.
过长的varchar: `InnoDb`将其存储为`BLOB`. 

UTF-8字符集:
varchar(1000)存储空间: 未知,因为UTF-8比较复杂,每个字符空间不定;
最大存储空间: 每个字符最多3B,因此最大空间为2+3000=3002B; 

**tip**
对于数据'hello',varchar(100)和varchar(5)实际存储空间一致(如果是latin1,6B),
但使用varchar(5)在计算时能减少内存消耗. (尤其是使用内存临时表进行排序时)


- varchar的优点
与char相比节省存储空间,因为短的数据会占用更少的空间,不像定长会浪费. 

- varchar的缺点
变长行在update时可能需要额外工作,如果行变长,MyIsam会将行分拆成不同片段存储;
InnoDb会分裂页来使行可以放进页内.

**使用varchar的场景**
- max(列的长度) 远大于 avg(列的长度). 这样大部分数据只需要很少的空间.
- 列的更新很少. (碎片不是问题)
- 使用了像UTF-8这样的复杂字符集,每个字符都使用不同的字节数进行存储.

**使用char的场景**
- 所有数据长度很接近. (如密码的MD5值,长度完全一致)
- 列更新频繁.(没碎片)
- 非常短的列. (不用存储长度信息)

> char列中数据末尾的空格会被截断.还会根据排序需要填充空格.
(服务器层处理,与存储引擎无关)


**BLOB和Text**
BLOB家族:
- TinyBlob
- SmallBlob = Blob
- MediumBlob
- LongBlob

Text家族:
- TinyText
- SmallText = Text
- MediumText
- LongText

- 存储
Blob和Text均作为对象存储(行里只存储指针),Innodb会使用专门的外部存储区域来存储实际的值. 指针的存储空间为1~4个字节.

- 排序
对每列前缀排序,前缀长度为`max_sort_length`个字节.
可以通过配置`max_sort_length`调整排序性能;
或者写`order by substring(col,length)`.

- 索引
Blob和Text列上建索引时,不能使用列的全部长度.

- 性能
由于Memory引擎不支持Blob和Text(以下讨论基于这个假设),
如果查询需要隐式临时表,将会使用MyIsam的磁盘临时表(内存到磁盘,性能下降).

explain执行计划中的extra列中包含`Using temporary`,说明查询使用了隐式临时表.


**tips**
1. 避免使用Blob和Text类型;(过长varchar会变成text,因此varchar不能过长(>255))
2. 临时手动转换成字符串; // 在使用Text的地方写substring(col,length);

临时表从内存转化到磁盘的时机:
- 有`Blob`或`Text`列;
- 大小超过`max_heap_table_size`或`tmp_table_size`. 

**Enum(枚举)**
如果字符串列的取值固定,可以使用枚举代替字符串. (会影响排序规则)
ENUM: 存储时为整数,比较时为字符串,但又不是原字符串.(= =||)

个人感觉, 这件事不应当在Mysql层做,应该在应用层做,因此此节略过.

## 4.1.4 日期和时间类型
相关类型包括:(最多只能精确到秒,如果要存储微秒,直接用bigint吧)
1. DateTime: 1001年到9999年. 精度为秒. 8个字节整数. 
2. Timestamp: 1970年到2038年. 精度为秒. 4个字节整数.(才到2038年,根本不能用)

个人感觉还是BigInt好,干净利落. (8B)

## 4.1.5 位数据类型(技术上来说是字符串类型)
相关类型包括:
1.Bit. (避免使用,行为诡异,游离在字符串和数字间)
例如Bit(2)可以存储俩布尔值.最大是Bit(64).
MyIsam: 所有Bit列打包起来存储;
InnoDb,Memory: 每个Bit列使用一个恰当大小的整型存放.
服务器层: Bit列作为字符串类型检索,遇到数字上下文又变成数字.

// 也就是存储时是数字,使用时是字符串或数字.

2.SET: 保存多个布尔值.
可以使用Find_in_set()/Field()函数. 
- 缺点:
改变列定义的代价昂贵. 也无法在Set列上进行索引查找.

- 小结
> Mysql的位数据类型都不靠谱,还是使用整型吧.
自己在应用层进行位操作, 简单/清晰一些.

## 4.1.6 ID列的数据类型选择
通常使用整型. 确定数据范围即可. 

*可以考虑使用美团的分布式ID生成器.*

# 4.2 数据库设计原则
1. 避免太多的列; 服务器层需要缓冲存储引擎层的行,变长的行会引入转换开销.
2. 避免太多关联;
3. 恰当使用NULL. 大多数时候避免NULL列,有时候可以使用.(当异常值不好定义时)

# 4.3 范式与反范式
范式优点:
- 更新简单;
- 冗余少;
- 设计更优雅;
- 简单查询性能高.

范式缺点:
- 关联多,复杂查询性能低. 
- 复杂查询性能差.

反范式(nosql)优点:
- 关联少,复杂查询性能好.

反范式缺点:
- 经常需要distinct,group by操作,影响性能, 代码写起来烦. 
- 更新性能差.
- 更新逻辑复杂.

- 小结
适当程度使用范式,不迷信三范式.
特定需求可以单独定制相应的缓存表和汇总表,而不改变原有的设计结构.

## 4.4 缓存表和汇总表
对于某些检索需求,可以定制化得设计相应的缓存表和汇总表,而不用更改原有的设计.
如需要计算24小时内发送的消息数. 可以维护一个表.

## 4.4.1 物化视图
Mysql并不原生支持,需要插件(FlexViews).

## 4.4.2 计数器表
100个槽的计数器表:(预先插入100个0)
```sql
create table hit_counter(
 slot tinyint unsigned not null primary key,
 cnt int unsigned not null
) engine=innodb;
```
每次更新计数器可以随机选一个slot进行,这样可以有更高的并发性能.
查询时只需要聚合一下结果就好:
```sql
select sum(cnt)
```

## 4.5 Alter Table操作的优化
Mysql执行Alter操作流程: (大部分情况下)
1. 用新的结构创建一个空表;
2. 从旧表中查出所有数据插入新表.
3. 删除旧表.


Innodb的优化: 通过排序来建立索引.使建索引更快并有一个紧凑的布局.

大部分Alter操作导致Mysql服务中断. // 导致事务强制提交.
相应解决技巧:
1. 在一个不提供服务的机器上进行Alter,然后和提供服务的主库进行切换;
2. 影子拷贝. 复制一张新表,通过重命名交换两张表.

不会引起表重建的Alter操作:
- 更改或删除一个列的默认值.

方法1: // 重建全表// 可以通过Show Status查到.
```sql
Alter table xxx
MODIFY COLUMN duration tinyint(3) not null default 5;
```

方法2:// 直接更改元数据(`.frm`文件).
```sql
Alter table xxx
Alter Column duration Set Default 5;
```

修改列的方法包括: Alter Column,Modify Column,Change Column.

- `ALter Column`: 只能修改列的默认值.直接修改元数据,非常快.
```sql
alter table film alter column rental_duration set default 5;  
alter table film alter column rental_duration drop default;
```

- `Change Column`: 可以修改列的一切. 修改数据(重建表)
- `Modify Column`: 比change少一个重命名功能,其他一样.

### 4.5.1 只修改元数据(.frm文件)
适用场景:
- 移除一个列的`Auto_increment`属性;
- 增加移除更改ENUM和SET常量.

流程:
1. 创建一张相同结构的空表,进行相应修改;
2. 关闭正在使用的表,并禁止任何表被打开; (通过执行Flush tables with read lock. )
3. 交换`.frm`文件;
4. 释放第二步的锁.(执行Unlock Tables)

### 4.5.2 快速创建MyIsam索引
流程:
1. 禁用索引;// Alter table xxx Disable Keys;
2. 加载数据;// Load Data
3. 启用索引.// Alter table xxx Enable Keys. 

> 之所以能更快,是因为不用逐行更改索引了. 批量建立索引可以使用排序构建, 索引树碎片更少,更紧凑.

由于Disable Keys对于唯一索引无效(`Unique Key`),所以如果涉及到唯一索引,可以这么做:
1. 删除唯一索引;
2. 加载数据;
3. 重建唯一索引. 
// 这种方法下,唯一性需要自行保证. 应用场景: 备份数据, 对数据知根知底的情况下.


还有一种方法是通过交换元数据文件,如上一节中的做法.
然后通过`Repair Table`来重建索引, 个人觉得有些危险,还是删索引比较简单安全.





---- 
前面两章的内容:(需要网上资料配合实验)
# 第二章 Benchmark
# 第三章 服务器性能剖析
查看某条查询执行的时间分布:
```sql
set profiling=1;
-- select * from xxx ;
show profiles;
show profile for query 1;
```
查看某查询的执行计划:
```sql
explain select xxx;
```

sending data中的行为包括:
```
关联时搜索匹配的行
...
```