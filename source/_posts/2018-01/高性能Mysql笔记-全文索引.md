---
title: 高性能Mysql笔记-全文索引
date: 2018-01-14 19:33:44
tags: mysql
categories:
- mysql
---

# MyISAM的全文索引(第七章)
有很多限制,比较弱.
- 作用对象:
```
全文集合: 将需要索引的列全部拼接成一个字符串,然后进行索引.
```
由于是对列进行拼接,因此无法设置哪一列更重要.(没有权重)

- 实现:
```
B-Tree索引,两层.
第一层: 所有关键字;
第二层: 一组相关的文档指针. 
```
全文索引不会索引文档对象中的所有词语,过滤规则如下:
1. 停用词. 默认按通用英语的使用,可以通过参数ft_stopword_file指定;
2. 长度条件. ft_min_word_len <=len <= ft_max_word_len. 

## 7.10.1 自然语言的全文索引
文档对象和查询的相关度.
相关度= f(匹配的关键词个数,关键词在文档中出现的次数)

原理:
索引中出现次数越少=>相关度越高
常见单词=>相关度低.

创建全文索引语法:
```
ALTER TABLE film_text ADD FULLTEXT INDEX fulltext_article(title,description);
```


自然语言搜索的查询语法:
```
select film_id,title,Right(description,25)
,Match(title,description) against ('factor casualties') as relevance -- 相关度
FROM film_text
WHERE  Match(title,description) against ('factor casualties')
```
执行计划:
Mysql将搜索词语分为两个独立的关键词进行搜索. (`factory`和`casualties`)
搜索对象: `title`和`description`组成的列的全文索引.

## 7.10.2 布尔全文索引
编写布尔搜索查询时,可以通过一些前缀修饰符来定制搜索.
```
dino: 包含dino的行rank更高;
~dino: 包含dino的行rank更低;
+dino: 行记录必须包含dino;
-dino: 行记录不能包含dino;
dino*: 以dino开头的行rank更高.
本例中dino作为搜索词可能太短,其实太短的词并不会被全文索引,查询优化器也可能要求搜索词>=ft_min_word_len.
```

查询语法:
```
select film_id,title,right(description,25)
FROM film_text
where match(title,description)
Against('+factory +casualties' In Boolean Mode);
-- title拼接description后必须同时包含factory和casualties
```

## 7.10.3 插件和限制
可以加入插件改变:
1. 分词方式(如C++);
2. 预处理(如PDF).

Mysql全文索引判断相关性的方法: 词频;
限制: 
1. 全文索引能全部load到内存时,才能快.
2. 判断相关性只有词频,没有顺序;
3. 优化器会优先使用全文索引,而不管是否有其他更优索引, 并且全文索引会在其他索引之前使用. 
4. 碎片多.


# Sphinx + Mysql + SphinxSE (附录F)
Sphinx: 开源全文搜索引擎. 可将数据源配置为Mysql查询的结果. 可水平扩展.
SphinxSE: Mysql的Sphinx插件,以便支持将Sphinx的搜索结果和Mysql的表进行join. 

Sphinx性能:
1. 查询1GB时间:  10~100ms;
2. 单CPU处理能力:10~100GB. 

分布式搜索工作流程:
1. 向所有服务器发送远程查询;
2. 执行本地索引搜索;
3. 从每个服务器读取部分搜索结果;
4. 合并结果返回客户端.

## 架构
- Indexed
从不同数据源创建全文索引.作为后台任务,一般定时运行;
- Searchd
运行时服务于客户端,查询创建好的索引.

## SphinxSE可插拔存储引擎
建表语法:
```
create table search_table
(id int not null
,weight int not null
,query varchar(3070) not null
,group_id int
,INDEX(query)
) engine=Spinx conncet="sphinx://localhost:3312/test"
```
查询语法:(把查询藏在query里,传递给sphinx)
```
select * from search_table where query='test;mode=all';
```
`Searchd`服务返回查询结果,存储引擎把数据转换成mysql表.

与`ElasticSearch`比较:
https://zhuanlan.zhihu.com/p/21334385

决定倒向`ElasticSearch`社区.XD


