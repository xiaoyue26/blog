---
title: linux自连接
date: 2020-03-24 10:02:26
tags:
- tcp
categories: 
- 网络

---

linux支持自连接，也就是同一个ip，同一个机器，自己起一个server、client互相连接。
例如使用命令:
```shell
nc localhost $port -p $port
```
对应TCP协议中的合法设定，同时打开:
{% img /images/2020-03/open_now.png 800 1200 open_now %}

## 误触可能
虽然这是linux的一个feature，但是我们平时遇到这种情况一般是因为bug，也就是误触。

一种可能：
进程1: listen端口A，然后挂掉，释放端口A；
进程2: 源端口A, 连接端口A。

## 触发bug条件
listen端口选择了`net.ipv4.ip_local_port_range`范围；

`net.ipv4.ip_local_port_range`范围里的端口是linux连接目标端口时，选择源端口的范围。可以通过修改`/etc/sysctl.conf`调整。（如果要调大connect并发度）
```shell
// 查看:
cat /proc/sys/net/ipv4/ip_local_port_range
// 生效:
sysctl -p /etc/sysctl.conf 
```
因为connect使用的端口会从这个范围取，所以如果listen也用了这个范围，就很可能冲突，形成自连接。

### 并发度
内核3.2以前:  需保证`源ip,源端口`2元组不同,并发度更低; (随机选择端口)
内核3.2以后:  需保证`源ip,源端口,目标ip,目标端口`4元组不同即可,并发度更大； (顺序选择端口)


### 恶化bug条件
1. 使用连接池，自连接无法释放；
2. Linux内核>=3.10: 2.6是随机选择端口,3.10是顺序递增选择；
