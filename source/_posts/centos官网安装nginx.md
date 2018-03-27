title: centos官网安装nginx
date: 2014-10-29 12:00:55
tags:
- 配置
categories:
- 配置
---


`vi /etc/yum.repos.d/nginx.repo`
加入如下内容：
```yaml
[nginx]
name=nginx repo 
baseurl=http://nginx.org/packages/centos/6/$basearch/
gpgcheck=0
enabled=1
```

然后直接`yum install nginx`。
