title: centos6.4安装vnc
date: 2014-10-17 17:35:25
tags:
- 配置
categories:
- 配置
---


1. 安装vnc
`yum install tigervnc-server`

2. 安装vnc字体
`yum install libXfont`

3. 查看vnc安装
`rpm -q vnc tigervnc-server`
安装成功显示
`package vnc is not installed`
`tigervnc-server-1.1.0-8.el6_5.x86_64`

4、设置远程桌面用户 root 密码
```
vncpasswd
Password:   密码
Verify:   验证
```

5、把远程桌面的用户加入到配置文件中
`vi /etc/sysconfig/vncservers`

6、使用vi编辑器打开配置文件，在文件中添加下面两行命令
```
VNCSERVERS="1:root"             #指定远程用户
VNCSERVERARGS[1]="-geometry 1024x768 -nolisten tcp -localhost"        #指定远程桌面分辨率    
```

7、开启VNC端口
`vi /etc/sysconfig/iptables`
使用vi编辑器打开配置文件，在文件中添加下面一行命令(最好放上面一行)
`-A INPUT -m state --state NEW -m tcp -p tcp --dport 5901 -j ACCEPT`

8、重启防火墙
`service iptables restart`

9、手动开启vnc
`vncserver`
