---
title: HTTP method 规范
date: 2017-02-04 19:36:12
tags: 
- http 
- restful
categories: http
---

# HTTP method 规范

- 创建题目
/api/questions
POST


- 更新题目
/api/questions
PUT

- 删除题目
/api/questions
DELETE


- 获取题目
/api/questions
GET

# HTTP4种方法的用法规范
- POST(create) 
- PUT(createOrUpdate) 
- GET(get) 
- DELETE(delete)

> 解释一下PUT的createOrUpdate:
POST 和 PUT 都可以用于create, 不同的地方是PUT是指定id的, 如果该id对应的资源服务端已经存在则是update, 否则就是create. 而POST是不带id的， 每次POST都会创建一个新对象.

PUT的例子如收藏功能
所以PUT是幂等的， POST不是幂等的. 
另外DELETE也是幂等的.
GET, DELETE 不能带payload.


## 层次化url:
> GET /api/courses/12/questions/101 的接口表示获取课程Id=12 的一道Id=101的题目

## 服务端无request session

服务端不记录客户端一次会话过程的上下文信息, 如果业务上需要也是由客户端来记录上下文信息， 并在每一次请求中以参数(或cookie)的方式带上
所以服务端无状态
