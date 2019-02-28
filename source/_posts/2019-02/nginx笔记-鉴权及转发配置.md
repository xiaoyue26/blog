---
title: nginx笔记-鉴权及转发配置
date: 2019-02-28 15:10:31
tags:
- nginx
categories:
- nginx

---

学习资料:
http://blog.jobbole.com/tag/nginx/
https://segmentfault.com/a/1190000013267839#articleHeader0

鉴权模块的官方文档:
http://nginx.org/en/docs/http/ngx_http_auth_request_module.html


## 易混淆点
**URL尾部的`/`区别**
url分为`location`配置中的`url`和实际用户访问的`url`：

### 1. location中的url
无区别。末尾是否有`/`,含义一样。

### 2. 实际用户访问的url
也就是浏览器地址栏中的。

首先实际访问url的话：
(1)末尾有`/`: 表示目录，如`localhost/dir/`，服务器就会匹配目录下的默认文件（比如`index.html`）;
(2)末尾无`/`: 表示文件，如`localhost/file`.

**特殊情况:**
根目录。
直接访问域名，如访问http://www.xxx.com
，这个时候浏览器知道用户访问的肯定不是文件，而且服务器一般会配置`location /`这个配置项，所以访问根目录有没有`/`都一样。
**也就是以下两种访问url等效:** 
1. 访问http://www.xxx.com
2. 访问http://www.xxx.com/
浏览器请求的时候自动给第1种加上`/`变成第2种。

## nginx配置的逻辑
- 特点:
1. 声明式
nginx的配置文件是声明式的,因此不能用过程式语言来理解它。
换句话说:
并不是写在前面的就先执行.

2. 第三方模块
nginx有核心模块和第三方模块（插件）。
不同模块的配置可能写在一块儿,但执行顺序可能无关联,甚至互相影响。

举例一些模块的功能：
```
--with-http_dav_module: http文件管理;
--with-http_flv_module: flv流媒体支持;
--with-mail: 邮件支持;
--with-mail_ssl_module: 邮件加密;
--with-debug: debug日志支持;
--with-http_auth_request_module: 鉴权转发支持。
```

## 一种可能的鉴权转发配置：

```conf
#user  nobody;
worker_processes  3;

error_log  logs/error.log;
#error_log  logs/error.log  notice;
#error_log  logs/error.log  info;

# pid        logs/nginx.pid; # 这个必须停服改


events {
    worker_connections  2048;
}


http {
    include       mime.types;
    default_type  application/octet-stream;

    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  logs/access.log  main;

    sendfile        on;
    #tcp_nopush     on;

    #keepalive_timeout  0;
    keepalive_timeout  65;

    #gzip  on;
    gzip  on;
    gzip_min_length 1k;
    gzip_buffers    4 16k;
    gzip_http_version 1.0;
    gzip_comp_level 6;
    gzip_types text/plain text/css text/javascript application/json application/javascript application/x-javascript application/xml;
    gzip_vary on;

    # http_proxy 设置
    client_max_body_size   10m;
    client_body_buffer_size   128k;
    proxy_connect_timeout   75;
    proxy_send_timeout   75;
    proxy_read_timeout   75;
    proxy_buffer_size   4k;
    proxy_buffers   4 32k;
    proxy_busy_buffers_size   64k;
    proxy_temp_file_write_size  64k;
    proxy_temp_path   /usr/local/nginx/proxy_temp 1 2;

    upstream backend_hexo {
	    server localhost:8081 max_fails=2 fail_timeout=30s ;
    }

    server {
        listen       80;
        server_name  localhost;

        #charset koi8-r;

        access_log  logs/host.access.log  main;

        location / {
            auth_request /auth;
            add_header Cache-Control no-store;
            proxy_set_header Host $host;

            proxy_pass        http://backend_hexo;
            proxy_redirect off;
            # 后端的Web服务器可以通过X-Forwarded-For获取用户真实IP
            proxy_set_header  Host  $host;
            proxy_set_header  X-Real-IP  $remote_addr;
            proxy_set_header  X-Forwarded-For  $proxy_add_x_forwarded_for;
            proxy_next_upstream error timeout invalid_header http_500 http_502 http_503 http_504;
        }

        location = /auth {
            proxy_pass http://auth_server:8080/api/hexo_permission;
            proxy_pass_request_body off;
            proxy_set_header Content-Length "";
            proxy_set_header X-Original-URI $request_uri;
        }


        #error_page  404              /404.html;

        # redirect server error pages to the static page /50x.html
        #
        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   html;
        }
    }
}
```



