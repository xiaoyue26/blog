title: apt-get本地源
date: 2016-01-03 10:59:01
tags: 配置
categories:
- 配置
---


>上次安装ambari，由于网速问题下载包下载了好久。如果反复这样的话太浪费时间了，又找不到能用的国内镜像源，决定自己建一个本地源。

首先有一台服务器用官方的`ambari.list`源
```
sudo apt-get install ambari-server
sudo apt-get install ambari-agent
```
下载好了相关的`deb`包。
接下来要做的工作就是想要复用这些文件。

1.`apt-get`安装的软件都保存在`/var/cache/apt/archives/`目录下，所以先把它们拷贝到别的目录下：
```
sudo mkdir ~/data/soft -p 
sudo cp -p /var/cache/apt/archives/*.deb  ~/data/soft/
cd ~/data/soft/
#修改一下拥有者和组，方便之后缩小权限:
chown -R am-server ~/data/
chgrp -R am-server ~/data/
```
2.安装一下依赖扫描工具：
```
sudo apt-get install  dpkg-dev -y
```
3.生成`Packages.gz`
```
dpkg-scanpackages soft/ |gzip > soft/Packages.gz
```
4.nginx对于静态资源并发比较好：
```
# 安装nginx:
sudo apt-get install nginx
# 配置一下nginx的启动用户：
vi /etc/nginx/nginx.conf
# 把user从www-data改为自己的用户am-server
user am-server;
:wq 
vi /etc/nginx/sites-available/default
# 更改根目录和location:
server {
	listen   80; ## listen for ipv4; this line is default and implied
	#listen   [::]:80 default ipv6only=on; ## listen for ipv6

	root /home/am-server/data;
	index index.html index.htm;

	# Make site accessible from http://localhost/
	server_name localhost;

	location / {
		# First attempt to serve request as file, then
		# as directory, then fall back to index.html
		try_files $uri $uri/ /index.html;
		
		# Uncomment to enable naxsi on this location
		# include /etc/nginx/naxsi.rules
	}
	location /soft{
		autoindex on;
		allow all;
	}
}
这里我顺手把index文件也拷进了data/
cp /usr/share/nginx/www/index.html /data
#保存之后重启nginx即可：
service nginx restart
```
在浏览器上检查是否可以访问：
[http://10.2.14.120/][1]
[http://10.2.14.120/soft][2]

如果出现问题的话，可以通过检查两个log文件寻找原因：
```
# 访问日志：
cat /var/log/nginx/access.log;
# 错误日志：
cat /var/log/nginx/error.log;
```
有可能是权限等问题。

5.配置后本地源的服务器后，可以在其他机器上测试一下apt-get能否正常识别：
```
sudo vi /etc/apt/source.list.d
#添加以下内容：
deb http://10.2.14.120 soft/
# 保存退出后执行
sudo apt-get update
sudo apt-get install ambari-server
```
这个时候应该可以看到飞驰的下载速度了。


感动吧.XD

  [1]: http://10.2.14.120/
  [2]: http://10.2.14.120/soft
