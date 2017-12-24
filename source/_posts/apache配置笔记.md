title: apache配置笔记
date: 2014-11-11 11:31:44
tags:
- 配置
categories:
- 配置

---


`cat /etc/httpd/conf/httpd.conf`

整个日志分为三部分：
(1)全局环境
(2)主配置：
虚拟主机没有处理的请求该如何处理；同时也是虚拟主机配置的缺省值；
(3)虚拟主机的设置，接收发往不同IP或不同域名的请求，用同一个apache2服务器进程进行处理。
没有框在`<VirtualHost>`标签里的设置会覆盖默认设置，所以尽量把自定义设置放在标签里。


1. `HostnameLookups Off`
这个选项表示：日志记录来访者时，只记录IP。
如果`Off`改成`On`，那就每次要先去DNS查该IP对应的域名，太麻烦了。

2. `DirectoryIndex index.html index.html.var index.php`
当请求的地址对应的是一个目录时，apache就会在改目录下找index.html..等等。

3. `ServerRoot /etc/httpd`
此时，`logs/foo.log`(相对路径)被解释为`/etc/httpd/logs/foo.log`
绝对路径`/logs/foo.log`依然还是`/logs/foo.log`。
若NFS下运行apache,则此配置比较复杂，需要参考：(或再百度下研究下)
http://httpd.apache.org/docs/2.2/mod/mpm_common.html#lockfile

4. MPM：多道处理模块，没研究。
5. `User apache`
`Group apache`

设置apache运行的用户。我猜测改成root，访问权限就全开了,但是也危险了。


6. `DocumentRoot "/var/www/html"`
放置服务文档的目录, 默认状态下,所有的请求都以这个目录为基础, 但是直接符号连接和别名可用于指向其他位置.


7. `ServerAdmin  xxx@qq.com`
这个指令用来设置服务器返回给客户端的错误信息中包含的管理员邮件地址。便于用户在收到错误信息后能及时与管理员取得联系。

8. `ServerSignature On`
方便调试，显示服务器信息；
安全的话，关掉，改成`Off`。

9. 对于`Order deny,allow`
`Allow from all`的解释：http://zhanshenny.iteye.com/blog/1654332

10. `Options`这个子句用来说明一些主要的设置，目前可以使用的设置有`Indexes`，`Includes`，`FollowSymLinks`，`ExecCGI`，`MultiView`，当然还有两个最简单的选择就是`None`和`All`。
　　`None`是禁止所有选择，而`All` 允许上面的所有`Options`。一般我们主要关心的是`Indexes` 和`FollowSymLinks`。`Indexes` 是设定是否允许在目录下面没有`index.html` 的时候显示目录，而`FollowSymLinks` 决定是否可以通过符号连接跨越`DocumentRoot`。例如，尽管`/ftp` 不在`/home/httpd/html` 下面，但是我们仍然可以使用符号连接建立一个`/home/httpd/html/ftp`使得可以直接输入`http://mydomain.com/ftp` 来访问这个目录。
　　使用`FollowSymLinks` 的办法很简单，就是首先在合适的目录段落里面`Options Follow SymLinks`，（符号连接的上层就可以）然后建立一个别名：
　　`Alias /ftp/ "home/httpd/html/ftp/"`
　　后面的是你建立的到`/ftp` 的符号连接。注意这一行应该位于所有目录段落之外。

11. 虚拟主机配置：
http://man.chinaunix.net/newsoft/ApacheManual/vhosts/name-based.html
http://man.chinaunix.net/newsoft/ApacheManual/vhosts/examples.html
http://man.chinaunix.net/newsoft/ApacheManual/vhosts/mass.html
http://httpd.apache.org/docs/2.2/
`Listen`指令并不实现虚拟主机，它只是告诉主服务器去监听哪些地址和端口。 如果没有`<VirtualHost>`指令，服务器对所有请求一视同仁； 但是如果有`<VirtualHost>`，则服务器会作出不同的响应。 要实现虚拟主机，首先必须告诉服务器需要监听的地址和端口， 然后为特定的地址和端口建立一个`<VirtualHost>`段。 注意，如果`<VirtualHost>`段设置为服务器没有监听的地址和端口， 则此段无效。

12. 暂无需求，待续.
