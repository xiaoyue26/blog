---
title: sort命令笔记
date: 2017-11-28 20:51:29
tags: shell
categories: 
- shell
---

语法:
```shell
sort [OPTION]... [FILE]...
sort [OPTION]... --files0-from=F
```

参数:
```shell
-b : 忽略行首空格
-c : 检查文件是否已经按照顺序排列
-d : 忽略除了英文字母,数字,空格以外的字符
-f : 将小写字母视为大写字母
-i : 只考虑040到176之间的ASCII字符,其他的忽略
-m : 合并几个排序好的文件
-n : 按数字大小排列
-o : 输出文件
-r : 倒序排列
-t : 排序时的分隔符
+<起始列>-<结束列>: 排序列范围
```

示例1:
```shell
# 默认按第一列的ascii码排序
sort log.txt
```


示例2:
```shell
# 查看当前mysql线程中的各种状态的数量,按uv最多输出前10个:
pipe -e 'show processlist \G' | grep State: |  uniq -c | sort -rn | head -n10

```

