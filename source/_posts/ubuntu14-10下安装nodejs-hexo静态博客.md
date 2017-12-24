title: ubuntu14.10下安装nodejs_hexo静态博客
date: 2013-11-07 16:43:14
tags:
- 配置
categories:
- 配置

---


如果对版本没要求，直接安装：
```
sudo apt-get install nodejs
nodejs -v  #查看版本成功则安装成功
sudo apt-get install npm
```
或者
`curl http://npmjs.org/install.sh | sh `
权限不够的话:
`curl http://npmjs.org/install.sh | sudo sh`

为了防止node不被识别成nodejs，安装`nodejs-legacy`
```
sudo apt-get install nodejs-legacy
sudo npm install -g hexo
```
 
**出现graceful-fs问题的话：**
```shell
git clone git://github.com/isaacs/npm.git
cd npm/scripts
chmod +x install.sh
sudo ./install.sh
sudo npm install -g hexo
```
**找个地方开始建项目了：**
```shell
mkdir blog
cd blog
mkdir xiaoyue26
cd xiaoyue26
sudo hexo init  #sudo hexo init <folder>
sudo npm  install
sudo hexo generate
sudo hexo server
```
打开显示的ip进去看看。

(1)如果是用以前的博客的话，复制以前的博客备份xiaoyue.tar.gz,解压打开。添加新博客内容：先从作业部落下载自己写好的md文件,用notepad++转为utf-8编码。
然后rz进客户机中。跳到步骤(3).

(2)如果是第一次配置的话,从新开始配置:
改_config.yml最后三行：
```xml
deploy:
  type: git
  repository: git@github.com:xiaoyue26/xiaoyue26.github.io.git
  branch: master
```

> 其中也可以是repository: http://github.com/xiaoyue26/xiaoyue26.github.io.git 
> 可以省去github上添加ssh公钥的步骤。

然后命令行：
`sudo npm install hexo-deployer-git --save`
查看[github添加ssh公钥笔记][1]。
(3)
`hexo new post vagrant笔记`
撰写`source/__posts/vagrant笔记.md`中的正文内容。
```
hexo g
hexo d
```
两句可合并为`hexo d -g`
浏览器输入
[http://xiaoyue26.github.io][2]
访问博客。

---

1.hello.js:
```js
var http = require('http');
http.createServer(function(req,res){
        res.writeHead(200,{'Content-Type':'text/plain'});
        res.end('Hello World\n');
}).listen(1337);
//若写listen(1337,"127.0.0.1")则只能从本机访问；
//所以将后面的ip去掉。
console.log('Server running at http://127.0.0.1:1337');
```

2.查看端口相关：
查看本机的8080端口是谁在监听(或占用)
lsof -i:8080
查看正在连接的双方端口：
netstat (有所遗漏)
ss -t -a (有所遗漏)


  [1]: http://xiaoyue26.github.io/2013/09/28/github%E6%B7%BB%E5%8A%A0ssh%E5%85%AC%E9%92%A5/
  [2]: http://xiaoyue26.github.io
