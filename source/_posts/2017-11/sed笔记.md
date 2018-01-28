---
title: sed笔记
date: 2017-11-28 20:44:09
tags: shell
categories: 
- shell
---

语法:
```
Usage: sed [OPTION]... {script-only-if-no-other-script} [input-file]...
```

动作参数:
```
a: 新增
c: 取代
d: 删除
i: 插入
p: 打印
s: 取代
```


示例1:
```
# 在第四行后添加一行(内容为helloWorld),结果打印到console:
sed -e 4a\helloWorld log.txt
# 假如文件根本没有第四行,则不会添加.
```


示例2: (-e可以省略)
```
# 把文件的内容加上行号输出,并且删除第2,3行.
nl log.txt | sed '2,3d'

# 删除第3行及之后的行:
nl log.txt | sed '3,$d'
```
