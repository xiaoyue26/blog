---
title: nginx笔记-编译安装
date: 2019-02-28 15:08:23
tags:
- nginx
categories:
- nginx

---

参考资料：
https://www.jianshu.com/p/5eab0f83e3b4

# 架构
{% img /images/2019-02/master-worker.png 800 1200 master-worker %}
master+worker架构。

master: 管理`nginx.conf`,同步到worker;
worker: 单线程绑定cpu，实际处理/转发请求;

master咋同步配置到worker呢？
直接用新conf起新worker,旧worker处理完手头的活就kill掉。

# 常用命令
```shell
# 启动:
nginx -c nginx配置文件地址   
# 重新载入配置:
nginx -s reload
# 检查配置（或查看配置地址）：
nginx -t
# 停止：
nginx -s stop
```
比如如果想知道当前nginx的配置文件在哪里，可以运行`nginx -t`,就能看到了。

# 编译安装
下载解压缩: 
http://nginx.org/en/download.html
**配置**
```sh
export KERNEL_BITS=64
./configure --user=mengqifeng \
--group=staff \
--prefix=/usr/local/nginx \
--pid-path=/var/run/nginx.pid \
--lock-path=/var/lock/nginx.lock \
--with-http_ssl_module \
--with-http_dav_module \
--with-http_flv_module \
--with-http_realip_module \
--with-http_gzip_static_module \
--with-http_stub_status_module \
--with-mail --with-mail_ssl_module \
--with-debug \
--with-http_auth_request_module \
--http-client-body-temp-path=/var/tmp/nginx/client \
--http-proxy-temp-path=/var/tmp/nginx/proxy \
--http-fastcgi-temp-path=/var/tmp/nginx/fastcgi \
--http-uwsgi-temp-path=/var/tmp/nginx/uwsgi \
--http-scgi-temp-path=/var/tmp/nginx/scgi \
--with-pcre=/Users/mengqifeng/Public/build_home/pcre-8.42 \
--with-openssl=/Users/mengqifeng/Public/build_home/openssl \
--with-zlib=/Users/mengqifeng/Public/build_home/zlib-1.2.11
```
其中最后3行要看情况,先不加。
报错以后下载pcre和openssl,加上参数提供给nginx.
**编译**
```sh
make
```
**安装**
```sh
sudo make install 
```

最后设置一下环境变量: (`/etc/profile.d/nginx.sh`)
```sh
export NGINX_HOME=/usr/local/nginx
export PATH=$PATH:$NGINX_HOME/sbin
```
