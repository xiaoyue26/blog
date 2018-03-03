---
title: 高性能Mysql笔记-第五章(2)
date: 2018-03-03 17:11:25
tags: mysql
categories:
- mysql

---

## 5.3.7 使用索引扫描来做排序
Mysql生成有序结果的两种方法: (详见第7章)
1. 排序操作. 
2. 按索引顺序扫描.

// explain结果中的type,如果为`index`,则使用第二种方法,按索引顺序扫描.
按索引顺序扫描工作流程:(Innodb二级索引)
1. 顺序扫描索引,获取主键指针;
2. 回表查询相应主键的数据行.
如果索引是覆盖索引,则可以避免上述第二步,减少一次磁盘IO,并且避免大范围扫描时的大量随机IO.

**正确示例:**
**索引为(a,b,c)**
示例1:
```
select a,b,c from t
where a='xxx' ORDER BY b,c
```
组合where条件和order by条件,而且都是顺序(ASC). 此外如果组合后是索引的最左前缀,也是可以利用索引进行排序的.

**错误示例**
示例2:
```
select a,b,c from t
where a='xxx' order by b DESC,c ASC.
```
由于顺序和建立索引的不同,因此不能利用到索引进行排序.
如果需要按不同方向排序, 可以通过存储这列的相反数或者反转值, 满足这种需求.

示例3:
```
select a,b,c from t
where a>'xxx' order by b,c
```
由于a是范围查询,因此无法利用索引的其他列(b和c).


