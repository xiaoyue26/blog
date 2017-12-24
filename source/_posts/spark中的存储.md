title: spark中的存储
date: 2016-03-26 15:12:52
tags:
- spark
- storage
categories:
- spark
---


- #存储架构
{% img /images/storage_in_spark(1).png 400 600 存储架构 %}

>Spark是一种基于内存的运算框架。数据存储在worker节点中。对于每一个worker，worker Excutor有一个JVM，数据存储在JVM中。

--- 

- #故障恢复
{% img /images/storage_in_spark(2).png 400 600 故障恢复 %}

>当计算引擎的进程损坏，Cache 丢失，Spark只能重新启动并计算恢复数据。

--- 

- #Lineage(世代)
{% img /images/storage_in_spark(3).png 400 600 Lineage %}

>Lineage保证数据的Reliability。
- 当数据E丢失后：
1. 通过世代找到相应的之前数据，重新部署一个Job将数据重新计算。
2. 将数据在底层文件系统中备份。

---

- #数据流
{% img /images/storage_in_spark(4).png 400 600 Lineage %}

>计算数据在JVM的内存中存储一份，以保证较少的网络通信和读写。同时记录存储数据的世代(lineage)，当数据丢失时，基于世代将job重新运行，得到相应数据。

---

- #计算过程
{% img /images/storage_in_spark(5).png 400 600 Lineage %}

>Spark内部，单个executor进程内RDD的分片数据是用Iterator流式访问的。
Iterator的hasNext方法和next方法是由RDD lineage上各个transformation携带的闭包函数复合而成的。
该复合Iterator每访问一个元素，就对该元素应用相应的复合函数，得到的结果再流式地落地。空间复杂度为O(1)。

---

- #持久化

>`spark.local.dir`：可以设置Spark的暂存目录，包括映射输出文件和需要存储在磁盘上的RDDs。这个磁盘目录在系统上面访问速度越快越好。可以用逗号隔开来设置多个目录。默认值为'/tmp'。
`org.apache.spark.storage.StorageLevel`中5种持久化的级别.(`result.persist(StorageLevel.DISK_ONLY)`) 默认为`MEMORY_ONLY`。

| 级别          | 空间    |  时间   | 内存   | 磁盘  |备注    |
| :--------:    | :-----: | :-----:  | :-----: | :----: | :-----:|
| MEMORY_ONLY | 多     |少      | 使用   | 不使用|    -    |
| MEMORY_ONLY_SER| 少  | 多     |使用    | 不使用|     -   |
|MEMORY_AND_DISK |多   |中等    | 部分 | 部分| 如果数据在内存中放不下，则溢写到磁盘上 |
| MEMORY_AND_DISK_SER| 少 |多 | 部分 |部分| 如果数据在内存中放不下，则溢写到磁盘上。在内存中存放序列化后的数据。 |
| DISK_ONLY   | 少     | 多     | 部分   |部分   |    -    |

---

- #数据分享
{% img /images/storage_in_spark(6).png 400 600 数据分享 %}

>在Spark中，如果job2需要Job1运算的数据，Job1首先需要将数据写入到HDFS的block中，会产生硬盘甚至跨网络的读写，同时在HDFS中默认数据需要写三份，因此造成性能的损失。

