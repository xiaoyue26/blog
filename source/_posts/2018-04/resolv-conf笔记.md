---
title: resolv.conf笔记
date: 2018-04-08 11:37:38
tag:
- 配置
categories:
- 配置
---

`/etc/resolv.conf`配置文件的4个关键字:
```shell
// 1. 定义dns服务器的地址,按序查找
nameserver 8.8.8.8
nameserver 114.114.114.114

// 2. 本地域名.  
domain google.com

// 3. 搜索域名.
search google.com cn.bing.com

// 4. 返回域名的排序
sortlist
```

主要用到的其实是`nameserver`和`search`.

其中`search`的用例如下:
```shell
ping new
```
由于`new`这个域名找不到ip;
尝试搜索`new.google.com`,(`search`域的第一项)
如果还是搜不到,就尝试搜索`new.cn.bing.com`.
也就说`search`会尝试补齐域名。


至于`domain`,相当于配置`search`的默认值。



