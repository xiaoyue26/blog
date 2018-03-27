---
title: fluentd学习笔记
date: 2016-12-24 19:12:51
categories:
- 配置
---


## 配置文件
`vi /etc/td-agent/td-agent.conf`
## 插件
`cd /etc/td-agent/plugin`
## 日志
`tail -fn 100 /var/log/td-agent/td-agent.log`
## 运行
`sudo /opt/td-agent/usr/sbin/td-agent --log /var/log/td-agent/td-agent.log --use-v1-config`

## 测试
`curl -X POST -d 'json={"json":"message"}' http://localhost:8888/debug.test`

# 配置
`source`: 输入
`match`: 输出目标
`filter`: 过滤 (输入与输出之间)
`system`: 系统级别设置
`label`: 操作名字
`@include`: 引入其他文件

## fluentd Event事件的生命周期
- 设置:
> 定义输入\监听,匹配规则,输出.

- event的结构:
`(tag,time,record)`
input插件负责把数据源的数据转换到event的结构.(三元组) 
例如数据源是: 
`192.168.0.1 - - [28/Feb/2013:12:00:00 +0900] "GET / HTTP/1.1" 200 777`
- event:
```
tag: apache.access # set by configuration
time: 1362020400   # 28/Feb/2013:12:00:00 +0900
record: {"user":"-","method":"GET","code":200,"size":777,"host":"192.168.0.1","path":"/"}
```
## filter:
- 过滤器:
```xml
<filter test.cycle>
  @type grep
  exclude1 action logout
</filter>
```
这个filter会过滤掉这种输入:
```shell
curl -i -X POST -d 'json={"action":"logout","user":2}' http://localhost:8888/test.cycle
```

## label
```xml
<source>
  @type http
  bind 0.0.0.0
  port 8888
  @label @STAGING
</source>

<filter test.cycle>
  @type grep
  exclude1 action login
</filter>

<label @STAGING>
  <filter test.cycle>
    @type grep
    exclude1 action logout
  </filter>
  <match test.cycle>
    @type stdout
  </match>
</label>
```

## source type
几种输入type:
> forward: tcp
http: http


- `e.g.`:

```xml
# Receive events from 24224/tcp
# This is used by log forwarding and the fluent-cat command
<source>
  @type forward
  port 24224
</source>

# http://this.host:9880/myapp.access?json={"event":"data"}
<source>
  @type http
  port 9880
</source>
# 把发送到localhost:9880/test.cyle的数据转发到标准输出上.
<match test.cycle>
  @type stdout
</match>
# 发送到myapp.access的数据发到/var/log...文件.
<match myapp.access>
  @type file
  path /var/log/fluent/access
</match>
```