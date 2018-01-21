---
title: hive抽样
date: 2018-01-20 19:25:01
tags: hive
categories:
- hadoop
- hive

---


有时候需要快速估计值,可以借助抽样.Hive中抽样的方法:
# 抽取指定比例(如10%):
```
SELECT * FROM t_name TABLESAMPLE(0.1 PERCENT) s
```
# 抽取指定大小(如30M):
```
SELECT * FROM t_name TABLESAMPLE (30M)
```
# 块抽样,每个InputSplit抽取10行:
```
SELECT * FROM t_name TABLESAMPLE(10 ROWS) s
```
# 桶抽样:(要求原表是分桶表)
```
SELECT * FROM t_name_2 TABLESAMPLE(BUCKET 1 OUT OF 10 ON pcid);
```
# 一致性抽样:
上述抽样每次运行都是随机选取,所以结果每次都不同. 
如果要求每次抽样的结果是一样的,可以使用随机数发生器的伪随机性,进行系统抽样:
```
SELECT * FROM t_name
WHERE rand(unix_timestamp(concat(fdate,' ',ftime)))<=0.1 -- 抽取10%
```
上述代码中将表中的两列(fdate,ftime)转化为随机数发生器的种子.
如果想让每次抽样的结果不同,也可以将种子换成当前的时间戳,比较灵活.


抽样后,可以对样本进行计算,估计总体的相应统计量. 
如果是均值,可以用t检验获取总体的置信区间:
```
u±S/√n*t(1-0.5a,n-1)
```
如果是中位数,相应的置信区间公式如下:
```
上界位次: S1=(n-Z(a)*√n)/2
下界位次: n-floor(S1)+1
```

其他统计量的置信区间可以自行搜索,例如次序统计量: https://wenku.baidu.com/view/6526f76a7e21af45b307a861.html
