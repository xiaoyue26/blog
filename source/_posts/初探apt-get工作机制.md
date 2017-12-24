title: 初探apt-get工作机制
date: 2016-01-03 11:00:45
tags: 配置
categories:
- 配置
---


首先通过`apt-get update`命令，读取`/etc/apt/source.list`文件和`/etc/apt/source.list.d`文件夹下的`*.list`文件。文件中大部分是`http`服务器地址，以其中一个`archive`为例,安装`lrsz`时:
```shell
get http://cn.archive.ubuntu.com/ubuntu/ precise/universe lrzsz amd64 0.12.21-5 
```
通过读取服务器`http://cn.archive.ubuntu.com/ubuntu/dists/precise/universe/source/Sources.gz`文件，文件中书写了所有能下载的软件包的版本和依赖。
(`ubuntu12.04` 的即从`precise`目录下查找)
查找到包的实际路径是：
`pool/universe/l/lrzsz/lrzsz_amd64_0.12.21-5.deb`
即：
`http://cn.archive.ubuntu.com/ubuntu/pool/universe/l/lrzsz/lrzsz_amd64_0.12.21-5.deb`。


- 使用第三方源时经常遇到`gpg`证书认证通不过的情况,可以强行忽略危险：
```shell
apt-get --allow-unauthenticated update
```

