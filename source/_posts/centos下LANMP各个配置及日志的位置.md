title: centos下LANMP各个配置及日志的位置
date: 2014-10-03 13:22:16
tags:
- 配置
categories:
- 配置
---



1. 开机启动服务及自动联网:
`vi /etc/sysconfig/network-scripts/ifcfg-eth0`
把ONBOOT=no改为yes
**ifcfg-eth0配置示例:**
```
DEVICE=eth0
HWADDR=B8:AC:6F:34:AE:FD
TYPE=Ethernet
UUID=ae099b3e-bc3b-4907-8020-b618ef5c25b3
ONBOOT=yes
NM_CONTROLLED=yes
BOOTPROTO=static
#BOOTPROTO=dhcp
#BROADCAST=10.2.14.128
IPADDR=10.2.2.54
NETMASK=255.255.255.0
#NETWORK=10.2.14.0
GATEWAY=10.2.2.1
DNS1=202.112.128.51
#DNS2=10.2.0.251
HOSTNAME=pc54
```

2.
http://www.jb51.net/article/37987.htm

CentOS 中 Apache 的默认根目录在 `/var/www/html`

配置文件在`/etc/httpd/conf/httpd.conf`

其他配置存储在 `/etc/httpd/conf.d/`

查看mysql监听端口：
`netstat -tulpn | grep -i mysql`

CustomLog logs/access_log

apache日志在：
`cat /var/log/httpd/access_log`

nginx配置文件在：
`cat /etc/nginx/nginx.conf`
其他在 `/etc/nginx/conf.d/`下

使用命令查看nginx相关文件位置：
`rpm -ql nginx`
防火墙配置：
`vi /etc/sysconfig/iptables`
加入要放开的端口：
`-A INPUT -m state --state NEW -m tcp -p tcp --dport 3306 -j ACCEPT`
然后：
`service iptables restart`


nginx日志：
 `cat /var/log/nginx/host.access.log`


一了百了解决权限：
`setenforce Permissive `


```
location /phpmyadmin {
       index index.php;
 disable_symlinks off;
    }
```



