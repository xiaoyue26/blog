title: centos下nginx配置笔记
date: 2014-10-29 16:38:11
tags:
- 配置
categories:
- 配置
---


日志文件位置：
```
cat /var/log/nginx/host.access.log
cat /var/log/nginx/access.log
cat /var/log/nginx/error.log 
cat /var/log/nginx/NginxStatus.log
cat /var/log/nginx/StatusError.log
```
用户名：`webadmin`
密码： `nginx`

配置文件：
```
cat /etc/nginx/nginx.conf
cat /etc/nginx/conf.d/default.conf
```
root命令和alias命令使用：
http://blog.csdn.net/bjash/article/details/8596538

如果没有配置root或alias,nginx默认会去/etc/nginx中找，可通过error日志查看具体找的路径。
如果在server里配置了root(location外面)，则会去root命令配置的目录下寻找。

`location`命令总结：
http://blog.chinaunix.net/uid-25196855-id-108805.html

自己笔记：
1. 

```
location ~ .*\.(html|gif|jpg|jpeg|png|bmp|swf)$  {  
		root    /usr/share/nginx/html;  
		expires 30d;  
	} 
```

`~`表示下面将进行区分大小写的正则匹配；
**(注意`~`后面有一个空格)；**
`.*` 任意个任意字符；
`\.` (转义斜杠)一个点；
`(html|...)$`这些后缀名之一结尾。
所以其实可以简化为： `location ~ \.(html|gif|jpg|jpeg|png|bmp|swf)$`
反正前面都是任意，一堆废话。

2. `~` 区分大小写的正则；
  `~*`不区分大小写的正则；
  `^~`匹配优先级比较高的正则；
  
3. 注意写正则的时候先使用在线正则测试工具进行测试。

4. 规范的nginx配置示例: 
```
#使用的用户和组
user www www;
#指定工作衍生进程数（一般等于CPU的总核数或者总核数的两倍），每个进程耗费10MB-12MB内存
worker_processes 8;
#指定错误日志存放的路径，错误日志记录级别可选项为：[debug | info | noticd | warn | error | crit]
error_log  logs/error.log;
#指定错误日志级别
#error_log  logs/error.log  notice;
#error_log  logs/error.log  info;
#指定pid存放的路径，文件内记录当前nginx主进程的ID，kill -HUP 'logs/nginx.pid'
#pid        logs/nginx.pid;

#指定文件描述符数量
worker_rlimit_nofile 51200;

#工作模式及连接数上限
events {
    #提高linux的io操作选项，Linux系统推荐采用epoll模型，FreeBSD系统推荐采用kequeue，linux下建议开启
   use epoll;
   #允许最大连接数
   worker_connections      51200;
}

http {
    #mimie.types 浏览器请求的文件媒体类型
    include       mime.types;
    #用来告诉浏览器请求的文件媒体类型
    default_type  application/octet-stream;
    #设置使用的字符集，如果一个网站有多种字符集，请不要随便设置，应该让程序员在HTML代码中通过Meta标签设置
    #charset gb2312;
    
    #日志记录格式
    #log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
    #                  '$status $body_bytes_sent "$http_referer" '
    #                  '"$http_user_agent" "$http_x_forwarded_for"';
    
    #日志名称，和日志记录格式采用main
    #access_log  logs/access.log  main;
    
    server_names_hash_bucket_size 128;
    #用于设置客户端请求的Header头缓冲区大小，大部分情况1KB大小足够。不能超过large_client_header_buffers缓冲区大小的设置
    client_header_buffer_size 32k;
    #该指令用于设置客户端请求的Header头缓冲区大小，默认值为4KB。
    large_client_header_buffers 4 32k;
    
    #设置客户端能够上传的文件大小，默认为1m
    client_max_body_size 8m;
     
    sendfile on;
    #该指令允许或禁止使用FreeBSD上的TCP_NOPUSH,或者Linux上的TCP_CORK套接字选项。
    #tcp_nopush on;
    
    
    #keepalive_timeout  0该指令可以使客户端到服务器端的连接持续有效
    keepalive_timeout 60;
    #该指令允许或禁止使用套接字选项TCP_NODELAY,仅适用于keep-alive连接
    tcp_nodelay on;
    
    
    fastcgi_connect_timeout 300;
    fastcgi_send_timeout 300;
    #该指令用于设置upstream模块等待FastCGI进程发送数据的超时时间，默认值为60s；
    fastcgi_read_timeout 200;
    #该指令设置FastCGI服务器相应头部的缓冲区大小。通常情况，该缓冲区大小设置等于fastcgi_buffers指令设置的一个缓冲区的大小。
    fastcgi_buffer_size 64k;
    #该指令设置了读取FastCGI进程返回信息的缓冲区数量和大小。
    fastcgi_buffers 4 64k;
    
    fastcgi_busy_buffers_size 128k;
    fastcgi_temp_file_writer_size 128k;
    
    #开启gzip压缩,对网页文件、css、js、xml等启动gzip压缩，减少数据传输量，提高访问速度。
    gzip on;
    #该指令允许压缩的页面最小字节数，页面字节数从header头中的Content-Length中进行获取。
    gzip_min_length 1k;
    #设置系统获取几个单位的缓存用于存储gzip的压缩结果数据流。下面的设置代表16k为单位，按照原始数据大小以16k为单位的4倍申请内存。
    gzip_buffers 4 16k;
    #识别http的协议版本。
    gzip_http_version 1.1;
    #gzip压缩比，1 压缩比最小处理速度最快，9 压缩比最大但处理速度最慢（传输快但比较消耗cpu）
    gzip_comp_level 2;
    #匹配mime类型进行压缩，无论是否指定，“text/html”类型总是会被压缩的。
    gzip_types text/plain application/x-javascript text/css application/xml;
    gzip_vary on;
    #该指令定义了一个数据区，其中记录会话状态信息。  定义一个叫“crawler”的记录去，总容量为10MB，以变量$binary_remote_addr作为会话的判断基准（即一个地址一个会话）
    #limit_zone crawler $binary_remote_addr 10m;
    
    #允许客户端请求的最大单个文件字节数
    client_max_body_size 300m;
    
    #缓冲区代理缓冲用户端请求的最大字节数，可以理解为先保存到本地再传给用户
    client_body_buffer_size 128k;
    
    #跟后端服务器连接的超时时间_发起握手等候响应超时时间
    proxy_connect_timeout 600;
    
    #连接成功后_等候后端服务器响应时间_其实已经进入后端的排队之中等候处理
    proxy_read_timeout 600;
    
    #后端服务器数据回传时间_就是在规定时间内后端服务器必须传完所有的数据
    proxy_send_timeout 600;
    
    #代理请求缓存区_这个缓存区会保存用户的头信息以供Nginx进行规则处理_一般只要能保存下头信息即可
    proxy_buffers 16k;
    
    #同上 告诉Nginx保存单个用的几个Buffer最大用多大空间
    proxy_buffers 4 32k;
    
    #如果系统很忙的时候可以申请更大的proxy_buffers 官方推荐*2
    proxy_busy_buffers_size 64k;
    
    #proxy缓存临时文件的大小
    proxy_temp_file_write_size 64k;
    
    #缓存    proxy_temp_path和proxy_cache_path必须在同一个分区
    proxy_temp_path /data2/proxy_temp_path;
    #该指令用于设置缓存文件的存放路径，设置缓存区名称为cache_one ，内存缓存空间大小为200M，自动清除超过1天没有被访问的缓存数据，硬盘缓存空间大小为30GB
    proxy_cache_path /data2/proxy_cache_path levels=1:2 keys_zone=cache_one:2000m inactive=1d max_size=30g;
    
    
    #include指令，使用此指令，可以包含任何你想要包含的配置文件，支持文件名匹配。
    #include vhosts/*.conf;
    
    upstream php_server_pool{
        #weight=NUMBER ——设置服务器的权重，权重数高被分配访问数越高，默认权重1.
        #max_fails=NUMBER——在参数fail_timeout指定的时间内对后端服务器请求失败的次数，如果检测到后端服务器无法连接及发生服务器错误（404错误除外），则标记为失败。默认值为1.设置为0这关闭这项检查
        #fail_timeout=TIME——在经历参数max_fails设置的失败次数后，暂停的时间
        #down——标记服务器为永久离线状态，用于ip_hash指令
        #backup——仅仅在非backup服务器全部繁忙的时候才启动
        
        #upstream模块拥有以下变量：
        #    $upstream_addr:处理请求的upstream服务器地址
        #    $upstream_status:upstream服务器的应答状态
        #    $upstream_response_time:Upstream服务器响应时间（毫秒），多个响应以逗号和冒号分隔。
        #    $upstream_http_$HEADER:任意的HTTP协议头信息，例如:$upstream_http_host
        
        server 192.168.1.10:80 weight=1 max_fails=2 fail_timeout=30s;
        server 192.168.1.11:80 weight=1 max_fails=2 fail_timeout=30s;
        server 192.168.1.12:80 weight=1 max_fails=2 fail_timeout=30s;
    }
    
    upstream message_server_pool{
        #ip_hash指令能够将某个客户端IP的请求通过哈希算法定位到同一台后端服务器。但无法保证后端服务器的负载均衡，所以建议后端服务器能做到session共享来代替nginx的ip_hash方式。
        #且如果后端服务器有时要从Nginx负载均衡中摘除一段时间，你必须将其标记为“down”
        #ip_hash;
        server 192.168.1.13:3245;
        server 192.168.1.14:3245 down;
    }
    
    upstream bbs_server_pool{
        server 192.168.1.15:80 weight=1 max_fails=2 fail_timeout=30s;
        server 192.168.1.16:80 weight=1 max_fails=2 fail_timeout=30s;
        server 192.168.1.17:80 weight=1 max_fails=2 fail_timeout=30s;
    }
    
    #第一个虚拟主机，反向代理php_server_pool这组服务器
    server
    {
        #该指令用于设置虚拟主机监听的服务器地址和端口号。
        #listen127.0.0.1:8080;
        #listen 8000;
        #listen *:8000;
        #listen localhost:8000;
        listen 80;
        server_name www.yourdomain.com;
        
        #SSL加密浏览
        ssl on
        ssl_certificate www.yourdomain.com.crt;
        ssl_certificate_key www.yourdomain.com.key;
        
        location /
        {
            #限制下载速度256KB/秒
            limit_rate 256k;
            #如果后端的服务器返回502、504、执行超时等错误，自动将请求转发到upstream负载均衡池中的另一台服务器，实现故障转移
            proxy_next_upstream http_502  http_504 error timeout invalid_header;
            proxy_pass http://php_server_pool;
            proxy_set_header Host www.yourdomain.com;
            proxy_set_header X-Forwarded-For $remote_addr;
        }
        
        access_log /data1/logs/www.yourdomain.com_access.log;        
    }
    
    #第二个虚拟主机，反向代理message_server_pool这组服务器
    server
    {
        listen 80;
        server_name www1.yourdomain.com;
        
        #访问http://www1.yourdomain.com/message/***地址，反向代理message_server_pool这组服务器
        location /
        {
            proxy_pass http://message_server_pool;
            proxy_set_header Host $host;
        }
        
        #访问除了/message/之外的http://www1.yourdomain.com/***，反向代理php_server_pool这组服务器
        location /message/
        {
            #DNS解析服务器的IP地址，可以在IE 工具-Internet选项-连接-局域网设置-代理服务器 中设置代理服务器IP地址和端口
            resolver 8.8.8.8；
            #该指令用于设置被代理服务器端口或套接字，以及URI
            proxy_pass http://php_server_pool;
            #该指令可以设置哪些从后端服务器传送过来的文件被Nginx存储。on保持文件与alias或root指令设置的目录一致，参数off不存储文件
            #proxy_store /data/www$original_uri;
            proxy_store on;
            #该指令用于指定创建文件和目录的权限
            proxy_store_access user:rw group:rw all:r;
            #指定一个本地目录来缓冲较大的代理请求
            proxy_temp_path /data/temp;
            #该指令用于在URL和文件系统路径之间实现映射。
            alias /data/www;
            #该指令允许重新定义或添加Header行道转发给被代理服务器的请求信息中，它的值可以是文本，也可以是变量，或者是文本和变量的组合。
            #使用$host变量，它的值相当于服务器的主机名（如果使用域名访问，则该值为域名；如果使用IP访问，则该值为IP）。此外可以将主机名和被代理服务器的端口一起传递  $host:$proxy_port
            proxy_set_header Host $host;
            proxy_set_header X-Forwarded-For $remote_addr;
            
        }
        
        access_log /data1/logs/www.yourdomain.com_access.log;        
    }
    
    #第三个虚拟主机
    server
    {
        listen 80;
        server_name bbs.yourdomain.com *.bbs.yourdomain.com;
            
        location /
        {
            proxy_pass http://bbs_server_pool;
            proxy_set_header Host $host;
            proxy_set_header X-Forwarded-For $remote_addr;
            #禁止指定的IP地址或者IP段访问某些虚拟主机或目录
            #deny 192.168.1.1/24;
            #允许指定的IP地址或者IP段访问某些虚拟主机或目录
            #allow 192.168.1.0/24;
            
        }
        
        #不记录日志
        access_log off;        
    }                 
    
}
```
