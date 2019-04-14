---
title: 2019年的人们如何生成HTTPS证书
date: 2019-04-14 19:59:09
tags:
- https
- ssl
- nginx
categories:
- http

---

由于配置相关的教程总是有年限限制，过期就不能用了，本教程至少保证2019.4.14还可用。
环境: centos,nginx,chrome
备注: 可以避免chrome的`NET::ERR_CERT_COMMON_NAME_INVALID`错误。

**摘要**
> 从HTTP升级到HTTPS: 用openssl命令创建本地的CA,然后自签证书，然后配置到nginx中，最后信任一下本地CA即可。

# 需求
从HTTP协议升级到HTTPS。
用HTTPS可以防止会话内容被拦截解析,同时防止中间人攻击，防止他人伪装称你的网站，欺骗你的客户。
SSL协议的详细含义可以参见:
https://xiaoyue26.github.io/2018/09/26/2018-09/%E9%80%9A%E4%BF%97%E7%90%86%E8%A7%A3SSL-TLS%E5%8D%8F%E8%AE%AE%E5%8C%BA%E5%88%AB%E4%B8%8E%E5%8E%9F%E7%90%86/

# 现有架构
请求=>nginx服务器=>后端网站服务

# 改造思路/原理
原理上只需要让nginx负责SSL协议的部分即可，不需要动后端网站服务。
客户端发送HTTPS请求到nginx服务器，nginx服务器转发HTTP请求到后端网站服务。
（封装的思想，上层变动对底层HTTP服务透明）

所以我们只需要关心架构中的前半部分:
请求=>nginx服务器

再分解一下这部分的话:
用户=>浏览器(chrome)==https请求=>nginx服务器

**整个架构中我们需要修改的部分:**
1. nginx配置. 

是的，就这么一项。所以改造成本很低。
当然了，如果不想花钱买官方CA证书的话，也就是自己弄一个CA, 然后给自己的网站颁发证书的话，还需要改动用户浏览器的信任CA，那么需要修改的部分就增加一项了:
1. nginx配置;
2. 用户浏览器信任CA。 

这里的证书、CA是个啥概念呢？
证书: 就好比我们网站的身份证；
CA  : 就好比派发身份证的派出所。
本质上是一个信任传递、担保的过程，用户浏览器会默认信任几个官方的CA，只要官方CA承认的网站，信任传递一下，用户就可以也信任了。
参见下图可以通过chrome右键"检查"的`security`面板查看证书的详细信息。
{% img /images/2019-04/ca.png 800 1200 ca %}

所以如果花钱让官方CA帮我们签发证书的话，用户可以直接默认信任我们的证书；
而如果我们自己弄的CA的话，好比自己开的黑作坊，用户不可能直接信任黑作坊签发的身份证的，就需要修改用户浏览器配置了，加入我们的私人CA证书。


# 公网HTTPS
## 生成数字证书
可以参考:
http://www.ruanyifeng.com/blog/2016/08/migrate-from-http-to-https.html
从
https://www.gogetssl.com/
https://www.ssls.com/
https://sslmate.com/
购买SSL证书。

免费的:
https://certbot.eff.org/
可以用这个工具，选择转发服务器和操作系统，生成证书:
https://certbot.eff.org/lets-encrypt

## 配置nginx
把原来nginx配置中的:
```
listen       80;
```
改成:
```
listen 443 ssl;
ssl_certificate /usr/local/nginx/ssl/private/server.crt;
ssl_certificate_key /usr/local/nginx/ssl/private/device.key;
```
这里的service.crt就是数字证书了。
如果要支持http和https同时可以访问，就把`listen 80`再加上。

如果要强制https，即使访问了http也强制跳转https(一般都需要这样搞),可以增加rewrite配置:
```
server {
        listen       80;
        server_name localhost;
        rewrite ^(.*) https://$server_name$1 permanent;
}
```

# 局域网HTTPS
公网https起码要买个域名,买个服务器(阿里云),如果只是局域网玩玩、或者自签证书,可以如下操作:
1. 本地生成一个CA;
2. 用这个CA给自己网站的数字证书签名，生成网站数字证书;
3. 修改nginx配置;
4. 配置用户chrome信任第一步中的CA。

可以看出多了1，2两步来生成证书,代替购买证书;
多了第4步来强制用户信任非官方CA.

## 1. 本地生成CA
找个干净的目录开始操作:
```
openssl genrsa -des3 -out rootCA.key 2048
openssl req -x509 -new -nodes -key rootCA.key -sha256 -days 1024 -out rootCA.pem
```
里面可以填下密码，email地址和ca的名字。其他的可以留空。

第一条命令: 生成本地CA的密钥`rootCA.key`(要记住你设置的密码,比如我的是`staythenight`)；
第二条命令: 用这个密钥进行签名，生成一张CA的证书`rootCA.pem`.
(这里设置的过期时间为1024天).

## 2. 生成网站数字证书(用这个CA给自己网站的数字证书签名)
为了避免chrome的`NET::ERR_CERT_COMMON_NAME_INVALID`错误，需要在网站证书里填一些额外的信息。
首先创建文件`server.csr.cnf`:
```
[req]
default_bits = 2048
prompt = no
default_md = sha256
distinguished_name = dn

[dn]
C=US
ST=RandomState
L=RandomCity
O=RandomOrganization
OU=RandomOrganizationUnit
emailAddress=296671657@qq.com # 修改成自己的email
CN = kandiandata.oa.com # 修改成自己的域名

```

然后创建文件`v3.ext`:
```
authorityKeyIdentifier=keyid,issuer
basicConstraints=CA:FALSE
keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
subjectAltName = @alt_names

[alt_names]
DNS.1 = kandiandata.oa.com # 修改成自己的域名
DNS.2 = localhost
```

创建证书:
```
openssl req -new -sha256 -nodes -out server.csr -newkey rsa:2048 -keyout device.key -config server.csr.cnf

openssl x509 -req -in server.csr \
-CA rootCA.pem \
-CAkey rootCA.key \
-CAcreateserial -out server.crt -days 1800 -sha256 -extfile v3.ext
```
第一条命令: 用`server.csr.cnf`配置生成网站证书`server.csr`,同时生成网站私钥`device.key`(给nginx用的)。
第二条命令: 用CA私钥`rootCA.key`以CA的名义(`rootCA.pem`)️给网站证书签名，生成CA签名后的证书`server.crt`，同时加上`v3.ext`中的配置（防止chrome报错）。


到这里我们就准备好了下一步nginx要用到的两个文件:
```
server.crt
device.key
```
`server.crt`: 网站的数字证书;
`device.key`: 网站的私钥，用来解开用户发过来的通信密码。详细原理参见:
http://xiaoyue26.github.io/2018/09/26/2018-09/%E9%80%9A%E4%BF%97%E7%90%86%E8%A7%A3SSL-TLS%E5%8D%8F%E8%AE%AE%E5%8C%BA%E5%88%AB%E4%B8%8E%E5%8E%9F%E7%90%86/

## 3. 配置nginx
这里和之前的一样,打开ssl支持，监听443:
```
listen 443 ssl;
ssl_certificate /usr/local/nginx/ssl/private/server.crt;
ssl_certificate_key /usr/local/nginx/ssl/private/device.key;
```
加上监听80+重定向:
```
server {
        listen       80;
        server_name localhost;
        rewrite ^(.*) https://$server_name$1 permanent;
}
```


## 4. 配置用户chrome信任第一步中的CA
将第一步中的`rootCA.pem`发送给用户，让它安装即可。
(千万不要发错了。)
如果是mac系统，可以直接双击安装到钥匙串中:
{% img /images/2019-04/rootca_pem.png 800 1200 rootca_pem %}
在钥匙串中选择`系统`=>`证书`,然后完全信任ca的证书即可:
{% img /images/2019-04/permit_ca.png 800 1200 permit_ca %}

最后得到chrome的承认:
{% img /images/2019-04/chrome_safe.png 800 1200 chrome_safe %}
