---
title: mysql可重复读下用for update导致的死锁
date: 2020-06-08 09:45:39
tags: mysql
categories: mysql

---

# 背景
试图用代码(JVM进程)来维持某列的唯一性。

# 实例
以下代码在可重复读隔离级别下执行，表中原来只有1~10的rank。假如A、B两个事务同时试图插入rank为21和22的rank；
```java
@Transactional
    public void createBannerDeadlock(Banner banner) {
        banner.recordCreate();
        List<Banner> rankList = cscCenterBannerDAO.selectForUpdate(banner.getRank());
        // 无记录,A占据间隙锁rank(10,+), B占据间隙锁 (10,+)，gap锁之间不互斥
        if (!rankList.isEmpty()) {
            throw new RuntimeException("记录已存在");
        }
        try {
            logger.info("睡5秒钟");
            Thread.sleep(MILLIS);
        } catch (InterruptedException e) {
            throw new RuntimeException("中断");
        }
        logger.info("开始插入: {}", System.currentTimeMillis());
        cscCenterBannerDAO.insert(banner); // 都申请插入意向锁, 都需要等待别人的间隙锁释放；死锁
    }
```
1.如果rank上没索引:
两个事务第一个for update就会互相阻塞(锁全表)，串行执行，没有死锁，两个插入都成功；
2.如果rank上有索引(普通索引或者唯一索引):
两个事务第一个for update不会互相阻塞(锁区间10~正无穷)，并行执行，有死锁，到最后真正插入的时候，两者都需要插入意向锁，然后都等待对方的gap锁释放，死锁；等待很久以后，一个成功一个失败(一个事务被回滚)；


死锁异常:
```java
Deadlock found when trying to get lock; try restarting transaction;
```


# 原理
两个原因导致了有索引的时候反而会死锁:
1.select for update的间隙锁之间不互斥,两个事务都能获得gap锁;
2.插入意向锁需要等待gap锁释放;


间隙锁相关: http://xiaoyue26.github.io/2018/05/26/2018-05/MVCC%E4%B8%8E%E9%97%B4%E9%9A%99%E9%94%81-mysql%E6%8B%BE%E9%81%97/
意向锁相关: http://xiaoyue26.github.io/2018/12/24/2018-12/mysql%E6%84%8F%E5%90%91%E9%94%81/


## select for update的结果
即使是同样的语句，也有可能有不同的加锁结果。
1.如果有唯一索引,命中了唯一记录: 行锁,互斥锁;
2.如果有唯一索引,没命中: gap锁，另一个事务也可以获得这个gap锁，但是不能插入数据；(后续有死锁可能)
3.如果有普通索引,命中了记录: 行锁+gap锁;(后续有死锁可能)
4.如果有普通索引,没有命中记录: gap锁,和情况2相同;(后续有死锁可能)
5.如果没有索引,直接锁全表，互斥，直接阻塞别的事务。（mysql的行锁是依赖索引的，这一点和oracle锁在数据块上不同；
因此如果没有索引或者没有用上索引，mysql就只能加表锁了。）

可见如果where条件没有命中记录的时候,如果有索引，反而可能有死锁风险。
除了select for update,其他进行当前读的语句(如delete)也可以获得gap锁，因此也有同样的烦恼。

那么都有哪些语句能获得gap锁呢？

## 参考
https://dev.mysql.com/doc/refman/5.7/en/innodb-locking.html#innodb-gap-locks

## gap锁
首先当然要可重复读和串行读才有gap锁。
然后根据上面的参考资料，gap锁只有一个目标:
就是防止别的事务在这个区间插入数据。

因此不同事务可以获得同一个区间的gap锁，因为这与上述目标并不冲突。两个事务可以获得这个区间的gap-S锁，gap-X锁，都可以。这一点和意向锁有点类似，只不过意向锁是加在表上的，gap锁是加在区间上的。

gap-S锁和gap-X锁没有区别。(因此select in share mode和select for update获得的gap锁本质是一样的，想要无锁读，就直接用select进行快照读。)

select in share mode/select for update/update/delete这4种语句是可能获取gap锁的。因此这4种当前读如果没有命中记录，而且又用到了索引，就会给死锁埋下风险。


## RC的优势
RC下: 扫描过但不匹配的记录不会加锁，或者是先加锁再释放，即semi-consistent read；
RR下: 扫描过记录都要加锁。
 
 
## RC的缺点
1.RC有幻读;
2.RC需要搭配row模式binlog;

mysql5.0以前的statement模式binlog和RC搭配有bug，因此为了兼容性，一般默认隔离级别RR，binlog模式row。

因此:
RC搭配row模式binlog;
RR搭配statement模式binlog;

## RC和RR读的区别
>  RR下，事务在第一个Read操作时，会建立Read View
RC下，事务在每次Read操作时，都会建立Read View

所以:
读已提交: 总是读到最新提交的数据;
可重复读: 总是读到和第一次读的时候相同的数据,与事务开始时间无关。

## innodb的7种锁类型
参考这里: https://dev.mysql.com/doc/refman/5.7/en/innodb-locking.html

行的读锁 S
行的写锁 X
表级：读意向锁 IS
表级：写意向锁 IX
(意向锁只影响和辅助表级操作，毕竟行级操作直接定位到具体行了，不需要意向锁的帮助。)
记录锁: Record Lock: 在索引上加锁。
gap锁: 区间内阻止插入的锁,当前读可能触发。
next-key锁: 记录锁+gap锁；
插入意向锁: 同区间insert可以并发,只要不重复;
自增锁: 如果是严格增模式,自增id会导致事务串行;


## 查看当前锁的信息
InnoDB整体状态，其中包括锁的情况:
```sql
show engine innodb status
```
只看锁信息:
mysql8.0.1以前:
```sql
-- 有事务在等的锁:
select * from information_schema.innodb_locks;
```
mysql8.0.1以后:
```sql
-- 所有的锁:
SELECT * FROM performance_schema.data_locks
```
两者字段的对应关系: https://dev.mysql.com/doc/refman/8.0/en/innodb-locks-table.html


