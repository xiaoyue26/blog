title: win7及ubuntu下安装解压版mysql
date: 2014-03-07 17:47:43
tags:
- 配置
categories:
- 配置

---


1. win7:
官网下载解压mysql后，在bin目录执行，
```
D:\MySQL\Server\mysql-5.6.20-win32\bin>mysqld -install
Service successfully installed.
net start mysql
mysql -u root 
update mysql.user set password=PASSWORD('root') where User='root' ;
flush privileges ;
```

停止mysql数据库服务:
```
net stop mysql
```
2. ubuntu:
```
sudo apt-get install mysql-server
sudo apt-get install mysql-client
sudo apt-get install libmysqlclient-dev
sudo netstat -tap | grep mysql
mysql -u root -p 
show databases;
use mysql ；
show tables ；
```
参考链接:
http://michael-wong.iteye.com/blog/976381
