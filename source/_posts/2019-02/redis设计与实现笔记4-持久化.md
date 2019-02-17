---
title: redis设计与实现笔记4-持久化
date: 2019-02-17 21:31:49
tags:
- redis
categories:
- redis

---

1. RDB: 定期(比如五分钟)做一次快照。 (`Redis DataBase`)压缩的二进制文件;
2. AOF: 类似于WAL，存每条操作指令日志。(`Append only file`)

题外:
`mmkv`: 内存映射文件(`MMF`)，场景是频繁刷盘，对丢失敏感前提下的尽量高性能。

## 相关命令
1. `SAVE`: 直接停服保存；
2. `BGSAVE`: 开一个子进程在后台保存。
检测到`RDB`或`AOF`文件后,redis进程会自动载入文件。(优先`AOF`,因为`AOF`一般更新更频繁)
{% img /images/2019-02/load_aof.png 800 1200 load_aof %}

`BGSAVE`执行期间，不接受类似指令如`SAVE`,`BGSAVE`,`BGWRITEAOF`。
载入文件期间，服务器阻塞不接受任何指令。

### 自动BGSAVE
```redis
save 900 1 # 900秒内至少1次修改
save 300 10 # 300秒内至少10次修改
save 60 10000 # 60秒内至少10000次修改
```

## 实现
```c
struct redisServer{
    // 记录自动bgsave的条件
    struct saveparam *saveparams;
    // 修改计数器:
    long long dirty;
    // 上一次执行保存的时间(unixtimestamp):
    time_t lastsave;
}
```
有了上述3个变量，可以保存需要的状态；
再加上每100ms执行的`serverCron`函数，就能完成自动保存的工作了。

# RDB持久化
## RDB文件结构(第6版RDB格式)
{% img /images/2019-02/rdb_struct.png 800 1200 rdb_struct %}
1. `"REDIS"`: 常量字符;
2. `db_version`:  4B,版本号；
3. `database`:所有保存的数据库（kv数据）,长度不定;
4. `EOF`: 1B，表示数据结束；(`377`)
5. `check_suim`: 8B，前4个部分校验和。

### database部分结构
{% img /images/2019-02/rdb_database.png 800 1200 rdb_database %}
1. `SELECTDB`: 1B,常量，区别于EOF；
2. `db_number`: 1B,2B,5B不等，具体的数据库号；
3. `key_value_pairs`: kv数据、过期时间，长度不等。

### key_value_pairs部分结构
1. EXPIRETIME_MS(可选)：1B,过期时间标记，常量;
2. `ms`(可选): 8B,具体过期时间(毫秒);
3. `TYPE`: 1B,类型，标记value的具体类型;
4. `key`: 总是字符串编码；
5. `value`: 值的编码根据`TYPE`字段判断。

其中`TYPE`的取值,一种对象类型或底层编码：
```
string,list,set,zset,hash
,list_ziplist,set_intset,zset_ziplist
,hash_ziplist
```
服务器根据这个`TYPE`字段决定如何解析`value`字段的编码。
 
### 实例
```shell
# 以ascii码查看:
od -c ./home/dump.rdb

0000000   R   E   D   I   S   0   0   0   6 377 334 263   C 360   Z 334
0000020 362   V


```
其中377表示EOF，`redis`5B,版本4B，`EOF`1B,校验和8B，总共18B.

```
# 以16进制查看:
od -x ./home/dump.rdb

0000000 4552 4944 3053 3030 ff36 b3dc f043 dc5a
0000020 56f2
```


# 第11章AOF持久化
三个步骤：
1. append：命令追加到redis的内存缓冲区;
2. 文件写入（不刷盘）:从redis的内存缓冲区到操作系统的内存页缓存;
3. fsync: 文件同步（实际刷盘）；
## 写入和同步
写入文件：实际上操作系统不会立即刷盘，而且处于性能先放内存缓冲区，后续再一起刷盘；
同步：调用`fsync`或`fdatasync`函数刷盘。


## 实现
```c
struct redisServer{
    // AOF缓冲区
    sds aof_buf;
}
```
1. 执行写命令;
2. 将写命令追加到aof_buf缓冲区的末尾；
3. 每次结束一个事件循环前，考虑是否将aof_buf内容写入同步到aof文件；（根据`appendfsync`选项）


### appendfsync选项
| appendfsync选项        | 操作   | 
| :--------:   | :----- |  
| always | 将aof_buf内容写入且同步到AOF文件     | 
| everysec(默认值)| 将aof_buf内容写入AOF文件，每秒同步一次  |  
|no | 将aof_buf内容写入AOF文件，同步由操作系统决定  |  
 
## AOF载入还原
{% img /images/2019-02/read_aof.png 800 1200 read_aof %}
用一个伪客户端执行所有aof命令即可。

## AOF重写(AOF文件压缩)
由于AOF文件记录的是命令而不是实际数据，可能会非常膨胀。
为了减少空间，同时加快载入恢复速度，可以根据实际数据进行压缩，跳过合并多余的命令。
方法类似于RDB，通过读取实际的数据库状态，生成AOF命令。

### 性能优化
出于对缓冲区的保护，如果遇到数据库中数据量太多的集合、有序集合、列表、哈希表，会让每条命令插入的数据量<=`REDIS_AOF_REWRITE_ITEMS_PER_CMD`(默认为64)。

### BGREWRITEAOF细节(尽量不停服)
{% img /images/2019-02/bg_rewrite.png 800 1200 bg_rewrite %}
调用子进程进程AOF重写期间，并不停服，为了保证一致性，会把这期间的命令追加到两个缓冲区：AOF缓冲区、AOF重写缓冲区。

重写完成后，stop the world:
1. AOF重写缓冲区=>AOF重写文件;
2. AOF重写文件替换旧AOF文件；(利用文件系统的改名原子性)
3. 恢复服务。
