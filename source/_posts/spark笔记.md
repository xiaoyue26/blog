---
title: spark笔记
date: 2017-04-24 20:04:34
tags: 
- spark 
- scala
categories:
- spark
---

https://github.com/databricks/learning-spark/blob/master/bin/fakelogs.sh
# scala语法相关
- val
> value,值. 定义时立即求值. (饿汉求值). 只求一次.

- var
> variable,变量.  定义时立即求值. 可改变赋值. 只求一次.

- def
> define. 每次使用时才求值.(惰性求值). 求N次

- lazy val
> 懒求值. 第一次使用时求值,但只求一次.

- 退出cli
> :q 或 sys.exit

- 查看版本:
> scala --version

# scala中文乱码问题:
http://www.runoob.com/w3cnote/mac-scala-chinese-show.html




#spark-submit作业提交相关:
1. 提交`python`和`jar`都是使用`spark-submit`命令;
2. `--master`可以接受的值:
```
spark://host:port
mesos://host:post
yarn
local
local[N]
local[*]
yarn-cluster 
```
当一直申请不到资源的时候,要使用`yarn-cluster`.


3. 格式:
```
spark-submit [options] <app jar|python file> [app options]
```

4. `
5. `可以接受的值:
```
client : 驱动程序放到本地机器
master : 驱动程序放到集群上
```
其他参数:
```
--class 运行java或scala程序时的主类
--name  显示的应用名
--jars  需要上传并放到应用classpath中的jar包的列表
--files 需要放到应用工作目录中的文件列表(如数据文件)
--py-files 需要添加到pythonpath中的文件,可以包含.py,.egg以及.zip文件
--executor-memory 执行器进程使用的内存量,字节为单位,可指定后缀
--driver-memory   驱动器进程使用的内存量,字节为单位,可指定后缀
```








```
$ZEP_SPARK_HOME/bin/spark-submit --master yarn-cluster $SPARK_HOME/examples/src/main/python/wordcount.py /user/fengmq01/test/input.txt
```


# spark streaming

## DStream
DStream由很多RDD组成,每个时间段的数据构成一个RDD.
```
val lines = ssc.socketTextStream("dx-pipe-cpu1-pm", 9092)
这里的lines类型是:
ReceiverInputDStream
从TCP输入流获取行,创建DStream.
```