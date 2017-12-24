title: ubuntu松耦合配置jdk和终端快捷键
date: 2015-10-03 21:46:15
tags:
- 配置
categories:
- 配置

---

 - **终端快捷键：**
新做系统后，为了提高工作效率，安装右键打开终端快捷键：
```
sudo apt-get install nautilus-open-terminal
nautilus -q
```
注销后登录，可以右键+e,在当前路径打开终端，比`ctrl+alt+T`顺手一点。而且`ctrl+alt+T`只能在`~`路径下打开终端，不够灵活。

 - **jdk:**
首先搜索jdk后进入官网下载最新版64位jdk。(多用tab快捷键补全。)
```shell
tar zxvf jdk(alt+/)
sudo mkdir /usr/lib/jdk/
sudo  mv jdk1.8.0_45/ /usr/lib/jdk/jdk1.8.0_45
cd /usr/lib/jdk/
sudo ln -s jdk(tab) jdk_current 
cd /etc/profile.d
sudo vi java.sh
export JAVA_HOME=/usr/lib/jdk/jdk_current
export CLASSPATH=.:%JAVA_HOME%/lib/dt.jar:%JAVA_HOME%/lib/tools.jar
export PATH=$PATH:$JAVA_HOME/bin
source java.sh
```
注销后登录, `java`和`javac`命令可以使用。这样做有几个好处:


- 更换jdk时,只需将jdk_current软链接指向新的jdk路径即可。
- 与别的环境变量耦合度较低，以此类推每个新的环境变量可以下`/etc/profile.d`路径下建立新的`hadoop.sh`，linux系统启动时会自动加载此目录下的`*.sh`文件。
- 可以把`java.sh`文件和jdk文件夹拷贝到别的电脑的`/etc/profile.d`路径下以添加jdk的环境变量。

- **配置root用户图形界面登录**
```
vi /etc/lightdm/lightdm.conf
# 添加：
greeter-show-manual-login=true
allow-guest=false
```

- **自动挂载新增的磁盘**
```
# 安装gparted
sudo apt-get install gparted
# 以root用户进入图形界面
# gparted配置分区, 查看ssd磁盘的uuid，然后修改文件以自动挂载新增的盘：
vi /etc/fstab 
# 末尾添加一行：
UUID=8436117e-d922-452d-8d86-56edb76b46f6 /home/xiaoyue26/ssd            ext4    defaults              0       2
# 上述格式为: UUID  挂载点 文件系统 策略 <dump> <pass>
```

default参数说明按照默认格式挂载.
其他参数：
```
auto: 开机自动挂载 
noauto: 开机不自动挂载 
defaults: 按照大多数永久文件系统的缺省值设置挂载定义 
ro: 按只读权限挂载 
rw: 按可读可写权限挂载 
user: 任何用户都可以挂载 
user: 同步磁盘与内存中的数据，async 则是异步 
```
> dump处为1的话，表示要将整个文件系统里的内容备份；
现在很少用到dump这个工具，在这里一般选0。0表示不做dump备份，1表示要进行dump备份，2也表示要做dump备份，不过，该分区的重要行比1小。

> pass用来指定如何使用fsck来检查硬盘。如果这里填 0，则不检查；挂载点为 / 的（即根分区），必须在这里填写 1，其它的都不能填写 1。
如果有分区填写大于 1的话，则在检查完根分区后，接着按填写的数字从小到大依次检查下去。同数字的同时检查。

- **退出保存,检查语法是否正确并且自动挂载盘：**
```
mount -a 
# 挂好以后, 递归更改一下权限：
chmod -R 775 /home/xiaoyue26/ssd #更改读写权限
chown -R xiaoyue26  /home/xiaoyue26/ssd #更改拥有者
chgrp -R xiaoyue26  /home/xiaoyue26/ssd #更高所属组

```


