---
title: sqoop学习笔记
date: 2017-09-28 20:08:36
tags:
---

https://geosmart.github.io/2016/02/24/Sqoop%E4%BD%BF%E7%94%A8%E7%AC%94%E8%AE%B0/
https://sqoop.apache.org/docs/1.4.6/SqoopUserGuide.html#_mysql
http://stackoverflow.com/questions/25887086/sqoop-export-using-update-key
# 1.列出数据库:
```
sqoop list-databases --connect jdbc:mysql://pipe-reader:3306/ -username pipe -password pipe
```

# 2.列出表:
```
sqoop list-tables --connect jdbc:mysql://pipe-reader:3306/pipe_tutor --username xxx --password pipe123
```

# 3. 复制表结构:
```
sqoop create-hive-table --connect jdbc:mysql://pipe-reader:3306/pipe_tutor --table dim_tutor_teacher \
--username xxx --password xxx \
--hive-table tutor.dim_tutor_teacher  --fields-terminated-by "\t"  --lines-terminated-by "\n";
```

# 4. 复制数据: 
```
sqoop import --connect jdbc:mysql://pipe-reader:3306/pipe_tutor \
--username xxx --password xxx \
--table dim_tutor_teacher --hive-import --hive-table tutor.dim_tutor_teacher \
-m 1 --fields-terminated-by "\t";
```

## 工作流程为:
>(1) 连接ZK
(2) 使用 `SELECT t.* FROM `dim_tutor_teacher` AS t LIMIT 1' 测试连接
(3) 生成一个运行jar包:
`/tmp/sqoop-maintain/compile/a2b05194f5b67b58688dff76426380ef/dim_tutor_teacher.jar`
(4) 提交jar包到RM,输出文件到 `/user/maintain/dim_tutor_teacher`
(5) load数据到内部表, `/user/maintain/dim_tutor_teacher`
下的文件除了`_success`文件都移动到 :
`/user/hive/warehouse/tutor.db/dim_tutor_teacher`

## 注意事项:
> (1) 若输出文件夹已经存在则会报错: `dfs -rmr  /user/maintain/dim_tutor_teacher`
(2) 内部表可以原先不存在,或者存在.
(3) 若表中原有数据,则load数据后会有两份数据. (需要自己手动删原有数据: `truncate table tutor.dim_tutor_teacher` )
(4) 使用`--direct`参数可以更快,但不支持blob和clob类型字段.
(5) 若hive表结构存在且与mysql表结构不一样,不影响sqoop工作(多退少补NULL).

## 启发:
>若使用外部表:
(1) 不需要删表数据,只需要和原来一样,删除输出文件夹.
(2) 表结构更新:
(2.1)内部表使用alter语句,外部表可以使用drop,create语句.
(2.2)若有dt且表结构中间多了一列,数据错位,无论内部表外部表,历史数据都不可直接用.需要写脚本转换调整.


# 5. 将hive中的表数据导入到mysql数据库表中
``` 
sqoop export --connect jdbc:mysql://pipe-reader:3306/pipe_tutor \
--username xxx --password xxx \
--table test_teacher --export-dir /user/hive/warehouse/tutor.db/dim_tutor_teacher  \
--input-fields-terminated-by '\t' \
--update-mode allowinsert --update-key id \
--input-null-string "\\\\N" --input-null-non-string "\\\\N" 
```

>注意事项:
1. 其中mysql表`pipe_tutor.test_teacher`需要自己手动建立表结构,这一点和从mysql导到hive不同.
2. 若`pipe_tutor.test_teacher`中原有一部分数据,则当出现主键冲突时,会卡死.使用`--update-mode allowinsert --update-key id`参数可解开卡死.
3. 导出的时候最好在末尾加上:
```
--input-null-string "\\\\N" --input-null-non-string "\\\\N"  
```
4. 更新模式:--update-mode updateonly/allowinsert
updateonly则只更新,allowinsert则对于没有冲突的primary key 进行插入.

# 6. --query 语句和--append使用 (mysql -> hdfs)
```
sqoop import --append --connect jdbc:mysql://pipe-reader:3306/pipe_tutor --username xxx --password xxx --query "select * from test_teacher where \$CONDITIONS and id>30"  -m 2  --split-by id --target-dir /user/maintain/dim_tutor_teacher --fields-terminated-by "\t";
``` 
或:
```
sqoop import --append --connect jdbc:mysql://pipe-reader:3306/pipe_tutor --username xxx --password xxx --query "select * from test_teacher where \$CONDITIONS and id>30"  -m 1  --target-dir /user/maintain/dim_tutor_teacher --fields-terminated-by "\t";
```
\$CONDITIONS 作为关键字,用于均摊任务到mapper中时填充筛选条件.
使用--query时必须使用\$CONDITIONS.

# 7. --columns  --where 语句使用(mysql -> hdfs)
```
sqoop import --append --connect jdbc:mysql://pipe-reader:3306/pipe_tutor --username xxx --password xxx --table test_teacher --columns "id,status,course"  --where "id < 3 "  -m 1  --target-dir /user/maintain/dim_tutor_teacher --fields-terminated-by ",";
```
--fields-terminated-by默认以','分隔.

导入hdfs:
--target-dir /user/maintain/dim_tutor_teacher
导入hive表:
 --hive-import --hive-table tutor.dim_tutor_teacher
 
分隔符:
--fields-terminated-by '\t'  --lines-terminated-by '\n' --optionally-enclosed-by '\"'     \

#8. 测试 --direct 参数

clob的字段是导入到hdfs上是正常显示文本，blob是二进制文件导出到hdfs上显示为16进制. 16进制转string:
http://www.aboutyun.com/thread-14008-1-1.html

对于clob和blob字段,使用--direct参数时应使用--query参数,不能使用--table参数.
或者不使用--direct参数.

参数:
```
--driver com.mysql.jdbc.Driver
```
语句:
```
sqoop import --append  \
--connect jdbc:mysql://mysql-ape-exercise13-reader:3306/ape_exercise13 \
--username ape --password ape@ytkz7c \
--query "select * from exercise where \$CONDITIONS   limit 1" \
-m 1  --target-dir /user/maintain/dim_tutor_teacher \
--direct \
 --fields-terminated-by "\t";
```

```
sqoop import --append  \
--connect jdbc:mysql://mysql-ape-conan-task4-reader:3306/ape_conan_task14 \
--username ape --password 87mPiRJky \
--query "select * from task where \$CONDITIONS" \
 -m 1  --target-dir /user/maintain/dim_tutor_teacher \
--direct \
 --fields-terminated-by "\t";
```

#9. sqoop执行文件:
sqoop --options-file ./test.opt
sqoop命令格式是 `sqoop CMD [args]`
从下文示例可以看出:
test.opt文件第一行是`CMD`(`import`);
以换行分隔参数(`args`).每对参数和值分成两行,
key前加`--`,value则不用.
关键字`$CONDITIONS`前的反斜杠不能加.
`--query`的`sql`语句的引号可有可无.
分隔符的`\t`语句的引号可有可无.

```test.opt
import
--append
--connect
jdbc:mysql://mysql-ape-conan-task4-reader:3306/ape_conan_task14
--username
ape
--password
87mPiRJky
--query
select * from task where $CONDITIONS
-m
1
--target-dir
/user/maintain/dim_tutor_teacher
--direct
--fields-terminated-by
\t
```


# 10. --warehouse-dir参数:
```
sqoop import  --append --connect jdbc:mysql://pipe-reader:3306/pipe_tutor \
--username xxx --password xxx \
--table dim_tutor_teacher \
--warehouse-dir /user/maintain/dim_tutor_teacher \
-m 1 --fields-terminated-by "\t";
```
`--warehouse-dir`会在目录建立与表名同名的目录(如果没有).然后把数据导入相应目录中.
日志:
```
17/05/25 10:22:35 INFO util.AppendUtils: Creating missing output directory - dim_tutor_teacher
```
`--target-dir`则直接写入,不创建目录.

# 11. 指定参数:
不预测执行(降低attempt数量,降低数据库压力).
```
-D
mapreduce.map.speculative=false
-D
mapreduce.job.name=question_hour
```
