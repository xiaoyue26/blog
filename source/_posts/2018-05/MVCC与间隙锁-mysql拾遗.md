---
title: MVCC与间隙锁-mysql拾遗
date: 2018-05-26 20:43:19
category: mysql
tags: mysql
---

## 参考资料
https://tech.meituan.com/innodb-lock.html

## 摘要
> 快照读(读): select语句
当前读(写): 增删改语句;特殊读(select * from table where ? lock in share mode).

> 快照读: 使用乐观锁(MVCC);
当前读: 使用间隙锁(其实是行锁+间隙锁=>nextKey锁)。

> 规范中的可重复读: 有幻读问题;
Mysql中的可重复读: 快照读没有幻读问题.

> RC: update只有行锁
RR(Mysql默认隔离级别): update where有间隙锁。（可能锁住不需要锁的范围）


之前一直对mysql中间隙锁有点迷糊, 不知道为什么有了MVCC这么好的东西,还需要间隙锁。
今天看了美团的技术博客才恍然大悟。

MVCC其实就是乐观锁:
优点: 并发性能好;
缺点: 实效性差,本质上读到的是快照(历史数据).

对于实效性要求比较高的写操作,MVCC是不可用的,比如:
```sql
update class_teacher set class_name='xxx' where teacherid=30;
```

这种操作如果要达到没有幻读,就要使用间隙锁.
间隙锁顾名思义,就是把teacherid=30的左边的间隙和右边的间隙都锁上,不能插入新的teacherid=30的。

如果现有teacherid总共有10,30,50三种，会对(10,50)都上锁,其中30的是行锁,[10,30),(30,50]的是间隙锁.
这个时候,不但30的插入不了,20的也插入不了了,因为正好在区间内。

特殊情况: 如果teacherid没有索引,那么也没什么间隙可以锁了,于是mysql就只好锁全表了.

> 以上说的间隙锁，只在RR(可重复读,mysql默认隔离级别下会有);
如果是RC，那只有行锁了，会出现幻读。

不符合规范的优化:
mysql加了范围锁以后,会把不符合条件的行锁提前释放.而这种操作是不符合两段锁规范的,两段锁协议要求流程要是一旦开始解锁就不能再加锁了。(所有解锁操作都在加锁后面.)