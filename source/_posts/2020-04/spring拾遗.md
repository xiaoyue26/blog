---
title: spring拾遗
date: 2020-04-19 08:58:37
tags:
- spring
- java
categories: 
- java
- spring


---

## 初始化顺序
如果有多个init方法,执行顺序:
1. `@PostContrust`注解标注的方法; 
2. 继承`InitialBean`后自己实现的`afterPropertiesSet`方法;
3. `@Bean(initMethod="xxx")`注解标注的方法;

## 销毁顺序
如果有多个销毁方法,执行顺序:(与上面类似)
1. `@PreDestory`注解标注的方法; 
2. 继承`DisposableBean`后自己实现的`destory`方法;
3. `@Bean(destoryMethod="xxx")`注解标注的方法;


## spring异常风格
全都用非受检(`runtimeException`),代码变得简洁,不需要到处声明异常、或`try catch`;
副作用: 使用者需要自己意识到会抛各种`BeanException`。

## spring偏向锁
由于需要使用兼容java5,使用了`synchronized`；
因此有偏向锁;
因此最好在main线程中初始化`applicationContext`，避免锁竞争，提高性能。

## 依赖注入
`@Autowired` 忽略静态字段;
接口回调注入: `implement xxxAware接口后`,会获得一个方法，可以从中得到、保存注入的对象。

强制的依赖: 构造器注入;
可选的依赖: setter注入; 字段注入; （时机、顺序不定、可循环）
配置类(声明类): 方法注入; `@Bean`

`@Autowired`的三个步骤:
``` 
1. 元信息解析: dependencyDescription;
2. 依赖查找: beanFactory
3. 依赖注入: 通过反射，注入到原字段。(所以很多都要求有setter)
```

## 提前注入
`@Bean`注解`static`方法时，会提前注入、加载。

## 依赖注入和依赖查找来源区别
依赖注入的来源默认会多4个，多4个spring自己默认注册的:
```
beanFactory
applicationContext
applicationEvenListener
ResourceLoader
```
这4个可以注入，但不能用`getBean`查找。
(只注册，不存到concurrentHashMap`)

大方向上，依赖注入4个来源：
1. 托管的bean; `definitionBean`, 启动以后不能注册; 
2. 单例对象; 启动以后还能注册;
3. 手动注册的`resolvedDependency`;
4. 外部化配置;

# prototype
spring不完全管`prototype`生命周期;
只管单例;
`prototype`销毁回调不执行,官方提到可以用`BeanPostProcessor`，但其实只能加初始化完成后的操作(此时大概率不需要销毁)，所以更合理的方式是用一个单例对象管理所有`prototype`(类似于领域里的`agg root`),在单例销毁时(实现`disposableBean`接口)，回调prototype的销毁;

单例和`prototype`均执行初始化回调。

# scope request
`@RequestScope`：
request scope对象: Ioc里是单例，但是返回给前端时是不同的。(快速销毁)
`@SessionScope`:
比request多一个锁。(避免同一个会话的并发)
同cookie时（同sessionid），返回前端的对象是相同的。(很长时间以后才销毁)
//tomcat默认session超时时间为30分钟，会序列化

jsp搜索范围：page-> request -> session -> servletContext

可以`implements Scope`自定义scope.

# IocBean初始化
1. beanDefinition加载;
2. 合并父类元信息;
3. 加载类; classLoader;
4. 实例化; `instantiation` , 赋值: `populate`
5. 返回前可以拦截替换代理对象;
6. 初始化
 

赋值前可以通过`postProcessProperties`增加需要赋值的字段。虽然名字是`post`但其实是`before`.

## Aware接口回调顺序
1. beanNameAware
2. beanClassLoaderAware
3. beanfactoryAware
4. environmentAware
5. EmbeddedValueResolveAware
6. ResourceLoaderAware
7. ApplicationEventPublisherAware
8. MessageSourceAware
9. ApplicationcontextAware

4-9是applicationContext有的。

# bean销毁
只是容器内销毁，并不意味着gc；

# beanPostProcesser
可以有多个，spring用list存储，所以先添加的先执行(FIFO)。

# 生命周期末期
applicationContext关闭
gc
调用finalized方法

# 生命周期完整
注册 
合并
实例化 前中后
赋值   前中后
aware回调 
初始化 前中后
销毁  前中后
gc
finalize


## 工具类
`ObjectUtils.nullsafeEquals`
`StringUtils.xxx`
`NamedThreadLocal`





## debug技巧
在底层打断点后，可以看方法调用栈。

## spring事务回滚
受检异常：不回滚；
非受检：例如`RuntimeException`回滚。

类内调用: 无事务(因为不经过代理加强)
类外调用：有事务

事务中新开线程： 新线程不能复用原连接(查不到未提交的数据)
解决方案：使用事务同步管理器:
```java
        TransactionSynchronizationManager.registerSynchronization(new TransactionSynchronizationAdapter() {

```


## mysql中立刻触发事务提交的语句
除了一些元数据操作，得注意新开一个事务也会触发原来的事务立即提交。
```sql
ALTER FUNCTION    
ALTER PROCEDURE    
ALTER TABLE    
BEGIN    
CREATE DATABASE    
CREATE FUNCTION    
CREATE INDEX    
CREATE PROCEDURE    
CREATE TABLE    
DROP DATABASE    
DROP FUNCTION    
DROP INDEX    
DROP PROCEDURE    
DROP TABLE    
UNLOCK TABLES    
LOAD MASTER DATA    
LOCK TABLES    
RENAME TABLE    
TRUNCATE TABLE    
SET AUTOCOMMIT=1    
START TRANSACTION    
```

## BeanFactory初始化

```
ResouceLoader加载配置信息
BeanDefintionReader解析配置信息，生成一个一个的BeanDefintion
BeanDefintion由BeanDefintionRegistry管理起来
BeanFactoryPostProcessor对配置信息进行加工(也就是处理配置的信息，一般通过PropertyPlaceholderConfigurer来实现)
实例化Bean
如果该Bean配置/实现了InstantiationAwareBean，则调用对应的方法
使用BeanWarpper来完成对象之间的属性配置(依赖)
如果该Bean配置/实现了Aware接口，则调用对应的方法
如果该Bean配置了BeanPostProcessor的before方法，则调用
如果该Bean配置了init-method或者实现InstantiationBean，则调用对应的方法
如果该Bean配置了BeanPostProcessor的after方法，则调用
将对象放入到HashMap中
最后如果配置了destroy或者DisposableBean的方法，则执行销毁操作
```


