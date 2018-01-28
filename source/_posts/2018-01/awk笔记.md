---
title: awk笔记
date: 2018-01-28 20:00:21
tags:
- awk
- shell
categories:
- shell
---

语法:
```
awk [参数] '具体命令' var=xxx file(s)
# 或
awk [参数] -f 具体命令所在文件 var=xxx file(s)
```

示例1:
```
# 每行按空格或\t分隔,输出文本的第1,4列
awk '{print $1,$4}' log.txt

# 比上面多了格式. (左对齐,第一列宽8,第二列宽10)
awk '{printf "%-8s %-10s\n",$1,$4}' log.txt
```
示例1中使用的是默认分隔符(空格或\t),也可以通过`-F`参数指定分隔符.


示例2:
```
# 使用逗号分隔
awk -F, '{print $1,$2}' log.txt
# 使用空格分隔
awk -F' ' '{print $1,$2}' log.txt
# 使用t字母分隔
awk 'BEGIN{FS="t"} {print $1}' log.txt
# 使用多个分隔符.
awk -F '[ ,]'  '{print $1,$2,$5}' log.txt
```

`awk -v #设置变量`
示例3:
```
awk -va=1 '{print $1,$1+a}' log.txt
# mac下的话:
awk -v a=1 '{print $1,$1+a}' log.txt
# 非数字+1=1
```

示例4:(过滤)
```
awk '$1>2' log.txt  # 第一列大于2
# 非数字大于2

# 第一列>2且第二列为Are:
awk '$1>2 && $2=="Are" {print $1,$2,$3}' log.txt  
```


示例5:(正则匹配)
~ 表示模式开始, 表达式首尾加上斜杠:
语法:
`awk '$2 ~ /正则表达式/ {print $1}' log.txt`

```
# 第二列符合正则: th[a-z]+ 
awk '$2 ~ /th[a-z]+/ {print $2}' log.txt

# 模式取反:
awk '$2 !~ /th[a-z]+/ {print $2}' log.txt`
```

awk脚本语法:
```
#!/bin/awk -f
#运行前
BEGIN {
    math = 0
    english = 0
    computer = 0
    printf "NAME    NO.   MATH  ENGLISH  COMPUTER   TOTAL\n"
    printf "---------------------------------------------\n"
}
#运行中 // 每一行
{
    math+=$3
    english+=$4
    computer+=$5
    printf "%-6s %-6s %4d %8d %8d %8d\n", $1, $2, $3,$4,$5, $3+$4+$5
}
#运行后
END {
    printf "---------------------------------------------\n"
    printf "  TOTAL:%10d %8d %8d \n", math, english, computer
    printf "AVERAGE:%10.2f %8.2f %8.2f\n", math/NR, english/NR, computer/NR
}
```