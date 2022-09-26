---
title: jwt破解
date: 2022-09-26 15:09:10
tag:
- 信息安全
- 网络攻防
- jwt
categories: 
- 网络
- 网络安全

---

# What: 什么是jwt
JSON Web Token（缩写 JWT）是目前最流行的跨域认证解决方案,主要应用于2个系统之间的授信场景。
也能用于身份token。
可以参考:
https://www.ruanyifeng.com/blog/2018/07/json_web_token-tutorial.html

可以使用https://jwt.io/查看jwt的三个组成部分:
{% img /images/2022-09/jwt.jpg 800 1200 jwt %}

# How: jwt的保密性如何？
jwt除了密钥，其他基本是明文，所以保密性较低。

可以用:
https://github.com/brendan-rius/c-jwt-cracker
对它进行破解。

或者直接：
```shell script
npm install --global jwt-cracker
jwt-cracker <jwt>
```
即可破解jwt中的密钥。

然后在jwt.io上修改payload，填上密钥重新加密即可篡改原来的内容实现破解。
