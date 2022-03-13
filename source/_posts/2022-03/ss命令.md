---
title: ss命令
date: 2022-03-13 10:02:13
tag:
- ss
- tcp
categories: 
- http

---


# ss命令
`Socket Statistics`
权威参考: https://man7.org/linux/man-pages/man8/ss.8.html
## 查看tcp连接的统计信息:
```shell script
ss -t | head
```
可以看到tcp连接的状态、收发情况、双方地址:
```shell script
State      Recv-Q Send-Q Local Address:Port                 Peer Address:Port
CLOSE-WAIT 1      0      10.28.234.170:64540                10.36.41.26:14326
```

## Sockets摘要信息:
```shell script
ss -s
```

## 查看已建立的http
```shell script
ss -o state established '( dport = :http or sport = :http )'
```

### 查看监听的端口
```shell script
ss -l
```

### 查看进程占用的端口
```shell script
ss -tnlp  | grep pid=97
```

### 状态过滤
```shell script
ss -4 state time-wait # 仅显示ipv4
ss -6 # 仅显示ipv6
```
状态的枚举可以通过 `ss -h`查看帮助来获取。

### 查看内存占用
用`ss -m`即可，具体输出的格式的定义如下：
```shell script
skmem:(r<rmem_alloc>,rb<rcv_buf>,t<wmem_alloc>,tb<snd_buf>,
                            f<fwd_alloc>,w<wmem_queued>,o<opt_mem>,
                            bl<back_log>,d<sock_drop>)

      <rmem_alloc>
            the memory allocated for receiving packet

      <rcv_buf>
             the total memory can be allocated for receiving
             packet

      <wmem_alloc>
             the memory used for sending packet (which has been
             sent to layer 3)

      <snd_buf>
             the total memory can be allocated for sending
             packet

      <fwd_alloc>
             the memory allocated by the socket as cache, but
             not used for receiving/sending packet yet. If need
             memory to send/receive packet, the memory in this
             cache will be used before allocate additional
             memory.

      <wmem_queued>
             The memory allocated for sending packet (which has
             not been sent to layer 3)

      <ropt_mem>
             The memory used for storing socket option, e.g.,
             the key for TCP MD5 signature

      <back_log>
             The memory used for the sk backlog queue. On a
             process context, if the process is receiving
             packet, and a new packet is received, it will be
             put into the sk backlog queue, so it can be
             received by the process immediately

      <sock_drop>
             the number of packets dropped before they are de-
             multiplexed into the socket
```
 









