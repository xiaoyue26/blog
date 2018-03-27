---
title: grep笔记
date: 2017-11-28 20:24:57
tags: shell
categories: 
- shell
---

grep语法:
```shell
grep [OPTION]... PATTEN [FILE]...
```

OPTION参数:
```shell
-i: 忽略大小写
-v: 反转查找
-w: 只显示全字符合的列
-c或--count: 计算符合范本样式的列数.
```

示例1:
```shell
# 计算处于init状态的mysql线程有多少个.
pipe -e 'show processlist \G' | grep -c 'State: init'
```

示例2:
```shell
# 查找目录/etc/acpi及其目录下所有文件中包含"update"的文件:
grep -r update /etc/acpi 
```