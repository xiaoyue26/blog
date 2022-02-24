---
title: 为什么先hash后加密的方法容易遭遇padding攻击
date: 2022-02-24 16:35:13
tags: 
- TLS
- SSL
- HTTP
- RSA
- AES
- HMAC
categories:
- http

---

# tls1.2实际加密方式
实际上加密算法是需要padding的，最早的padding方法是:
`AES(text+MAC(text)+padding)`
后来因为这种方式容易遭遇padding攻击，因此tls1.3采用了更安全的padding方法:
先`E=AES(text+padding)`
然后： `E+MAC(E)`

>padding攻击，通过反复修改部分内容、并触发解密过程，从而探测猜测加密算法的密钥；

# Why
`padding oracle attack`:
https://zh.wikipedia.org/wiki/%E5%AF%86%E6%96%87%E5%A1%AB%E5%A1%9E%E6%94%BB%E5%87%BB
从攻击方式可以看出，攻击者主要是以
1。修改最后1个字节；
2。触发解密，根据服务器返回（可能是耗时差异，也可能是返回错误），判断填充是否正确；
这种模式进行攻击。
所以如果在第2步，减少触发解密步骤的频率，则可以提供攻击难度。

回顾加密传输的两种处理步骤:
(1)hash->padding->加密;
(2)padding->加密->hash;

对应的解密步骤:
(1)解密->去掉padding->检查hash;
(2)检查hash->解密->去掉padding;

第1种方案：第一步就触发解密，容易被攻击；
第2种方案: 如果hash检查失败，就不会往后走解密逻辑了，因此攻击的难度提高了 。