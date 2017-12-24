title: ambari安装笔记
date: 2016-01-03 10:58:14
tags: 
- 配置
categories:
- 配置

---


> 终于闲下来可以学点新东西。学习一下[ambari][1]。
[http://www.linuxidc.com/Linux/2014-05/101531p2.htm][2]

- 首先弄四个干净ubuntu系统,安装ssh和jdk：
```shell
sudo passwd
root
sudo apt-get -y install openssh-server
# 这里掉坑里了,HDP2.2用的是jdk1.7,所以如果这里用了jdk1.8后面就只能安装HDP2.3了。以后试下jdk1.7。
tar zxvf jdk-8u60-linux-x64.tar.gz
sudo mkdir /usr/lib/jdk/
sudo mv jdk1.8.0_60/ /usr/lib/jdk/jdk1.8.0_60
cd /usr/lib/jdk/
sudo ln -s jdk1.8.0_60/ jdk_current 
cd /etc/profile.d
sudo vi java.sh
i
# ESC :wq
export JAVA_HOME=/usr/lib/jdk/jdk_current
export CLASSPATH=.:%JAVA_HOME%/lib/dt.jar:%JAVA_HOME%/lib/tools.jar
export PATH=$PATH:$JAVA_HOME/bin
source java.sh
```

- 挑一台安装`ambari-server`:
```
cd /etc/apt/sources.list.d
wget http://public-repo-1.hortonworks.com/ambari/ubuntu12/2.x/updates/2.1.2/ambari.list
apt-key adv --recv-keys --keyserver keyserver.ubuntu.com B9733A7A07513CAD
apt-get update
apt-get install ambari-server
#　这里下载起来很慢，可配置一个本地源，节省下次的时间。
ambari-server setup
ambari-server start
```


1.配置完成启动服务后，在同网段内一台机器的浏览器上访问：
`10.2.2.184:8080`
默认帐号密码都是`admin`，可以登录进去以后改密码。

2.创建集群前，需要配置主节点到所有节点免密码登录：下次可以试试`ssh-agent`，听说比手动配方便。
```shell
# 所有机器上运行: 配置自身无密码登录
ssh-keygen -t rsa 
cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys
 
# 集群之间共享公钥：
# 从第一台机器到最后一台机器执行一圈,最后把authorized_keys传回第一台机器
# 从第一台机器传到所有其他机器上。
# 第一台开始：
scp ~/.ssh/authorized_keys 10.2.2.185:~/.ssh/authorized_keys1
# 此后依次到最后一台(n-1次),一直到传回第一台：
cat ~/.ssh/authorized_keys1 >>~/.ssh/authorized_keys
scp ~/.ssh/authorized_keys 10.2.2.184:~/.ssh/authorized_keys
#第一台传到所有机器上：
scp ~/.ssh/authorized_keys 10.2.2.185:~/.ssh/authorized_keys
scp ~/.ssh/authorized_keys 10.2.2.188:~/.ssh/authorized_keys
scp ~/.ssh/authorized_keys 10.2.2.189:~/.ssh/authorized_keys

# master: try to login every slaver
ssh slave1
ssh 10.2.2.185
```
最后把`ambari-server`那台机器的ssh私钥粘贴到网页端。

3.首次尝试，所以配置的都是root用户。如果在slaver上创建了hadoop用户，安装的时候还是会提示让我们删掉，所以干脆还是用root先。
4.`ubuntu12.04.4`下自动安装失败，改为手动安装`ambari-agent`：
```shell
# 在每个slaver上：
sudo apt-get install ambari-agent
# 安装后以后
sudo ambari-agent start
# 然后回到网页端继续配置。
```
5.`host`需要配置完全。
```
vi /etc/hosts
# ... 略
sudo /etc/init.d/networking restart
```
6.`ntp`时间服务需要配置：
```shell
sudo apt-get install ntp
service --status-all | grep ntp #查看是否启动([+])
```
7.居然还要自己配置jdk...:(可能是因为master上自己配置了jdk,下次还是用它的。)
8.配置失败后重新配置的命令：
```
# master:
ambari-server stop
ambari-server reset
# slaver:
python /usr/lib/python2.6/site-packages/ambari_agent/HostCleanup.py --silent --skip=users
```
9.注册完成后,选择要安装的服务就可以慢慢等着安装好了，这一步也是在线安装所以很慢的。第一次可以少选一点服务，以后再慢慢添加服务，集群的节点也是可以动态扩展的。
10.默认配置：
ambari-server的pid放在：
/var/run/ambari-server/ambari-server.pid
日志输出在：
/var/log/ambari-server/ambari-server.out
错误日志：
/var/log/ambari-server/ambari-server.log
资源组织文件：
/var/lib/ambari-server/resources
数据库名：ambari
schema名：ambari
用户名：  ambari
密码：    bigdata
网页端的话用户名密码默认都是admin.
可以通过查看日志信息，寻找出错原因。
祝好XD~



  [1]: http://www.cnblogs.com/ivan0626/p/4143963.html
  [2]: http://www.linuxidc.com/Linux/2014-05/101531p2.htm
