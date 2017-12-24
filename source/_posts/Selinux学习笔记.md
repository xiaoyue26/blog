title: selinux学习笔记
date: 2014-10-07 17:28:30
tags:
- 配置
categories:
- 配置

---


1. 先安装工具:
```
yum install setools
yum install setroubleshoot
```

2. `selinux`是提供日志文件来记录错误信息的，错误信息记录在`/var/log/messages` 和 `/var/log/setroubleshoot/*` 里头，需要重启auditd服务来开启selinux的log服务：
`service auditd restart`

3. 
宽容模式：(只写日志)
`setenforce Permissive `
严格：
`setenforce Enforcing`
查看当前模式：
`getenforce`



4. audit日志超级长，而且看不懂；
先把它清空然后再现错误（只记录最新的错误）：
`echo >/var/log/audit/audit.log`
`sealert -a /var/log/audit/audit.log >./qq.txt`
把出错日志转化为人能看懂的内容
比如里面会提示你：
```
setsebool -P httpd_can_network_relay 1
setsebool -P httpd_can_network_connect 1
grep nginx /var/log/audit/audit.log | audit2allow -M mypol
semodule -i mypol.pp
```
可以试着照着做。一般能解决权限访问问题。


**关闭Selinux:**

`vi /etc/selinux/config`
```
#SELINUX=enforcing #注释掉
#SELINUXTYPE=targeted #注释掉
SELINUX=disabled #增加
```
`:wq  #保存，关闭`
`shutdown -r now #重启系统`



如果只是为了phpadmin则可以这样： 一条命令解决权限

`chcon -R -t httpd_sys_content_t /var/www/html/phpmyadmin`


权限：

`namei -om /var/www/html/phpadmin`
