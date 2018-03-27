title: ubuntu12.04下vncserver的安装
date: 2015-03-07 16:51:11
tags:
- 配置
categories:
- 配置

---


1. **安装：**
```shell
sudo apt-get install vnc4server
```
2. **启动：**
```shell
vncserver :1
```
**关闭：**
```shell
vncserver -kill :1
```

还需配置图形界面：
配置`~/.vnc/xstartup`：
```shell
unset SESSION_MANAGER
unset DBUS_SESSION_BUS_ADDRESS
# exec /etc/X11/xinit/xinitrc

# 中间部分配置不变

# x-window-manager &
export DESKTOP_SESSION=ubuntu-2d
# 这个 `ubuntu-2d`参考 `/usr/share/gnome-session/sessions/` 下的文件名设置成不同的值：
export GDMSESSION=ubuntu-2d
export STARTUP="/usr/bin/gnome-session --session=ubuntu-2d"
$STARTUP
```

**别漏了最后的$STARTUP。**
完整配置如下:
```shell
#!/bin/sh
# Uncomment the following two lines for normal desktop:
unset SESSION_MANAGER
unset DBUS_SESSION_BUS_ADDRESS
# exec /etc/X11/xinit/xinitrc
[ -x /etc/vnc/xstartup ] && exec /etc/vnc/xstartup
[ -r $HOME/.Xresources ] && xrdb $HOME/.Xresources
xsetroot -solid grey
vncconfig -iconic &
x-terminal-emulator -geometry 80x24+10+10 -ls -title "$VNCDESKTOP Desktop" &
#x-window-manager &
export DESKTOP_SESSION=ubuntu-2d
export GDMSESSION=ubuntu-2d
export STARTUP="/usr/bin/gnome-session --session=ubuntu-2d"
$STARTUP 

```

然后再启动vncserver就可以远程用vncviewer登录访问图形界面了。
**启动：**
```shell
vncserver :1
```

**关闭：**

```shell
vncserver -kill :1
```

例如win7下使用vncviewer，`vncserver`栏里输入ip和端口号`10.2.2.147:1`，其他保持默认,让vncserver选择。输入vncserver端启动`vncserver :1`时用户的密码。即可登录。

- vnc远程桌面屏幕大小：
在运行`vncserver`的时候可通过指定参数，在适合的屏幕下进行图形界面的访问。
```shell
vncserver -geometry 1600x900 -depth 24 :1
vncserver -geometry 1920x1080 -depth 32 :1
vncserver -geometry 1900x1000  :1
```
