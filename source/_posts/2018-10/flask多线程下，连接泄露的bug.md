---
title: flask多线程下，连接泄露的bug
date: 2018-10-23 16:35:52
tags: 
- mysql
- python
- flask
categories:
- mysql

---

# 架构图
{% img /images/2018-10/flask.png 400 600 flask %}

如图所示，底层使用mysql，web服务使用`flask-SqlAlchemy`的连接池（复用连接，减少创建销毁开销），逻辑层代码使用线程池（异步IO操作，如果要异步cpu操作，可以很方便改成进程池）。

# 基础知识
1. 使用`db.engine.execute(sql)`: 从连接池获取一个连接,执行完sql后自动`commit`;(`commit`操作的回调是: 归还连接到池里); 
2. 使用`session`的`orm`(`xxxModel.query等`): 默认配置及推荐配置是`autocommit=false`，执行完增删查改后，处于事务未提交的状态，也就是没有归还连接。如果要归还连接，可以使用语句:
```python
db.session.commit()
db.session.rollback()
db.session.close()
db.session.remove(): 底层会调用db.session.close()
```

小结:
> db.engine.execute(sql) => 自动commit => 自动归还
session(orm) => 手动commit => 手动归还
因此db.engine.execute(sql)是绝对安全的;
orm是有条件的。接着往下看orm的安全条件。

## 线程与session
使用flask的SqlAlchemy插件`flask-SqlAlchemy`时，每个线程可以直接用`db.session`获得session，即使不显式获得，使用orm的model时，其实也隐式得获得了`session`。

### 线程与session的关系: 
每个线程有自己的`threadlocal`的`session`对象，并且随着线程销毁，会自动释放`session`,也就是会隐式调用`session.remove`，也就是会隐式释放`session`的连接。

多线程两种使用:
1. `t1=threading.Thread(...)`;
2. 线程池: `future= pool.submit(...)`.
方法1的线程使用完以后自动销毁=>session自动销毁=>连接自动释放;
方法2的线程使用完以后归还线程池=>session手动销毁=>连接释放。

小结:
不使用线程池=>连接自动释放;
使用线程池=>连接手动释放.
手动释放的方法:
```
db.session.remove()
```

# 空闲连接超时与连接释放bug
前面说到使用线程池时，连接没有自动释放，一直维护在线程的threadlocal存储中(tls)。那么这样似乎也没有什么关系，只要线程池大小<连接池大小,这样连接池有空闲连接，每个线程也有自己的连接可以用，一切似乎也相安无事。

然而，这里有一个之前没有提到的机制：空闲连接超时回收。

### 空闲连接超时回收
#### mysql服务端:
定期检查现存连接的空闲时间，把超出`wait_timeout`的连接删除，此时客户端保存的长连接引用就失效了; 这个时间的设定:
```sql
show global variables like 'wait_timeout';
set global wait_timeout=10*60; -- seconds
```

#### web服务: 
flask会定期检查连接池里的连接，把空闲连接删除，重新向mysql服务端申请新的连接，这样就不会访问到失效的连接引用了。其中定期的时间是: `app.config['SQLALCHEMY_POOL_RECYCLE'] =xxx`(秒，应当设置为小于`wait_timeout`)。这就是为什么最好连接用完及时归还，否则可能就没法被flask刷新成新连接。


#### 空闲连接超时与连接释放bug
bug发生的流程
1. mysql服务端清除了空闲时间过长的连接;
2. 线程池中线程一直不销毁，因此持有了活了很久的session;
3. 活了很久的session持有了空闲很久的连接, 这个连接其实已经被服务端销毁了，因此已经不可用了，但是由于其一直没有归还到连接池中，因此一直没有得到更新。
4. 此时web服务收到数据请求，使用该线程中的该session中的连接，就会抛异常了，因为连接已经不可用了。

一般来说，空闲时间很长以后，线程池里所有线程的所有session的所有连接都会失效，因此就会完全无法通过orm访问数据库了。

相关异常信息:
```
 MySQL server has gone away
 Can't reconnect until invalid transaction is rolled back
```
这里之所以说`invalid transaction is rolled back`，是因为老session收到数据请求后，准备要用连接了。
而连接上的事务没有自动提交，也没有rollback，因此不能直接用。
因此尝试把连接上，上一次请求的事务提交，但由于连接已经失效，所以失败了。
