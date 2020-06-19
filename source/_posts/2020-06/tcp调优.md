---
title: tcp调优
date: 2020-06-19 17:44:55
tags: tcp
categories: tcp

---

# 原理：linux如何处理新的tcp连接
1. 触发时机: 客户端: 发起一个tcp连接（SYN）;

服务端: 
(相关参数: `net.ipv4.tcp_max_syn_backlog`)
TCP模块查看`max_syn_backlog`是否超阈值;
超阈值的话: 根据`tcp_abort_on_overflow`是丢弃还是reset;
未超的话: 放到半连接队列;

2. 触发时机: 客户端: 回复服务端ACK
服务端: 
(相关参数: `net.core.somaxconn`)
完全建立连接: 放到全连接队列; 

## 内核调优
1. `vim /etc/sysctl.conf` 在末尾添加：

```
net.core.somaxconn = 16384 # ESTABLISHED的连接队列最大
net.ipv4.tcp_max_syn_backlog = 65536 # SYN半开连接队列最大
net.core.wmem_default=8388608 # 默认发送窗口的字节大小,还有一个最大值的参数
net.core.rmem_default=8388608 # 默认接收窗口的字节大小,还有一个最大值的参数
```

2. `sysctl -p /etc/sysctl.conf`

3. `echo "1" > /proc/sys/net/ipv4/tcp_abort_on_overflow` # 满了以后显式发送RST包给客户端

4. 修改nginx配置
```
listen 80 backlog=16384;

listen 443 backlog=16384;
```
就是server里面的配置，在80 后面加上backlog=16384

5. `cd tnginx_1_0_0-1.0/bin/nginx -s reload`

6. 重启api服务(如果有)

7. 确认配置生效 `ss -lnt`

# 相关内核commit
since Linux 5.4 it was increased to 4096 https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=19f92a030ca6d772ab44b22ee6a01378a8cb32d4

查看现有配置:
```
sysctl -a  | grep somaxconn
```
