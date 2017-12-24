title: lanmp获取真实IP
date: 2014-10-13 16:30:44
tags:
- 配置
categories:
- 配置
---

```
wget https://github.com/ttkzw/mod_remoteip-httpd22/raw/master/mod_remoteip.c
/usr/local/apache/bin/apxs -i -c -n mod_remoteip.so mod_remoteip.c
vi /etc/httpd/conf/httpd.conf
```

加入：
`Include conf/extra/httpd-remoteip.conf`
在`/etc/httpd/conf`下创建`extra`文件夹
创建`httpd-remoteip.conf`文件，
内容如下：
```
LoadModule remoteip_module modules/mod_remoteip.so
RemoteIPHeader X-Forwarded-For
RemoteIPInternalProxy 127.0.0.1
```
然后
`service httpd restart`
