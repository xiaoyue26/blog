---
title: spring实战笔记
date: 2017-10-24 19:38:24
tags: 
- spring 
- java
categories: 
- java
- spring
---

# 第一章 主要特性
# DI 
- 依赖注入.
> 让Spring容器来管理对象的创建和销毁,注入;依赖关系更为明确,耦合程度降低.
以前是bean工厂,现在是各种ApplicationContext容器.

# AOP
- 切面.
> 声明切点,注入before,after事件,降低代码耦合.
```xml
<aop:config>
    <aop:aspect ref="minstrel">
      <aop:pointcut id="embark"
          expression="execution(* *.embarkOnQuest(..))"/>
        
      <aop:before pointcut-ref="embark" 
          method="singBeforeQuest"/>

      <aop:after pointcut-ref="embark" 
          method="singAfterQuest"/>
    </aop:aspect>
  </aop:config>
```

# 模板
```
jdbcTemplate+RowMapper, 简化jdbc查询.
// 用JPA的话就不用写RowMapper了.
```

## bean的生命周期
1. 实例化
2. 填充属性
3. setBeanName
4. setBeanFactory
5. setApplicationContext
6. 调用BeanPostProcessor的预初始化方法
7. 调用InitializingBean的afterProperties set方法
8. 调用自定义的初始化方法
9. 调用BeanPostProcessor的初始化后方法
10. ------ bean的使用
11. ------ 容器关闭
12. 调用DisposableBean的destroy方法
13. 调用自定义的销毁方法

# 第二章 DI
讲了一下声明和使用Bean的方法. 与Springboot实战相同,略过.

# 第三章 条件化配置\高级装配
各种profile切换,条件bean,配置注入等等,与Springboot实战相同,略过.

# 第四章 AOP
具体实现机制:
> Spring 采用jdk动态代理模式来实现Aop机制。Spring AOP采用动态代理过程：1.将切面使用动态代理的方式动态织入到目标对象，形成一个代理对象。2.目标对象如果没有实现代理接口，那么spring会采用CGLib来生成代理对象，该代理对象是目标对象的子类。3.目标对象如果是final类，也没有实现接口，就不能运用AOP

AspectJ是AOP的一个实现,Spring借鉴了AspectJ.(有合作)

## 术语
- Advice : 通知(增强)(Before,After,Around)
> 描述横切逻辑和方法的具体织入点（方法前、方法后、方法的两端等）

- PointCut: 切点(例子中的切点是execution表达式规定的方法)
> 指定在哪些类的哪些方法上织入横切逻辑.

- JoinPoint: 连接点 (粒度是方法)
- Advisor(切面): 
> 将Pointcut和Advice两者组装起来。有了Advisor的信息，Spring就可以利用JDK或CGLib的动态代理技术采用统一的方式为目标Bean创建织入切面的代理对象了

例子:
```xml
<aop:config>
    <aop:aspect ref="minstrel">
      <aop:pointcut id="embark"
          expression="execution(* *.embarkOnQuest(..))"/>
        
      <aop:before pointcut-ref="embark" 
          method="singBeforeQuest"/>

      <aop:after pointcut-ref="embark" 
          method="singAfterQuest"/>
    </aop:aspect>
  </aop:config>
```

- spring支持的AspectJ切点表达式(挺多的记不住,就不列全了):
```java
arg()
this
target
execution
...
```

用切点表达式来选取某个方法示例:
```java
execution(* concert.Performance.perform(..))
// * :  返回任意类型
// 方法所属的类是: concert.Performance
// 具体方法名是: perform
// 方法参数是: 任意参数(两个点..)
```

简单总结一下步骤(用注解):
1. 写一个config,把观众和演员都注册为Bean.加上`@EnableAspectJAutoProxy`.
2. 观众类上加`@Aspect`,写各种@Before,@After方法;


代码见:
https://github.com/xiaoyue26/spring-gradle/tree/master/src/main/java/com/xiaoyue/nov/practice/concert


书里还提供了一个思路,通过AOP扩展原来的类,而不改动原来的源代码.

## 原理总结
- 动态代理
>`JDK动态代理`：只能为接口创建动态代理实例，而不能针对类 。
`CGLib` :
（Code GenerationLibrary）动态代理：可以为任何类创建织入横切逻辑代理对象，主要是对指定的类生成一个子类，覆盖其中的方法，因为是继承，所以该类或方法最好不要声明成final。

- 原理对比：(尽量用接口,不用final)

JDK动态代理：(只能是接口)
> JDK动态代理技术。通过需要代理的目标类的getClass().getInterfaces()方法获取到接口信息（这里实际上是使用了Java反射技术。getClass()和getInterfaces()函数都在Class类中，Class对象描述的是一个正在运行期间的Java对象的类和接口信息），通过读取这些代理接口信息生成一个实现了代理接口的动态代理Class（动态生成代理类的字节码），然后通过反射机制获得动态代理类的构造函数，并利用该构造函数生成该Class的实例对象（InvokeHandler作为构造函数的入参传递进去），在调用具体方法前调用InvokeHandler来处理。


- CGLib动态代理：(不能是final类)
> 字节码技术。利用asm开源包，把代理对象类的class文件加载进来，通过修改其字节码生成子类来处理。采用非常底层的字节码技术，为一个类创建子类，并在子类中采用方法拦截的技术拦截所有父类方法的调用，并顺势织入横切逻辑。

- 两种配置切换:
1、如果目标对象实现了接口，默认情况下会采用JDK的动态代理实现AOP 
2、如果目标对象实现了接口，可以强制使用CGLIB实现AOP 
3、如果目标对象没有实现接口，必须采用CGLIB库，spring会自动在JDK动态代理和CGLIB之间转换

# 第二部分 WEB
# 第五章 构建
# 第六第七章 SpringMVC
内容,比较繁琐
# 第八章 Spring Web Flow
stackoverflow上有个帖子详细解释了为啥不能用这个框架.
https://stackoverflow.com/questions/29750720/what-are-the-spring-web-flow-advantages
包括:
1. xml配置难用;
2. 不直观;
3. 与第三方库(如模板引擎)整合复杂;
... // 看了前三个缺点就没耐心看后几个缺点了...
核心还是难用,比普通spring的xml配置复杂得多,几乎没有人能坚持看十分钟攻略.

# 第九章 安全 Spring Security
原理:
1. 用的filter. (AOP用的interpretor)

antMatchers用的是
ant pattern,和普通的通配符有点不一样,是专门用于url的通配符:
```
? 匹配任何单字符 
* 匹配0或者任意数量的字符 
** 匹配0或者更多的目录 
```
其实就是:
```
* 匹配任何字符，不含 “/”
** 匹配任何字符，含 “/”
```

匹配上路径以后,配置能够做什么.
```java
access("hasRole('READER')") 设置SpEL表达式为true时允许访问;
anonymous()  允许匿名访问
authenticated() 允许认证过的用户访问
denyAll() 拒绝所有
fullyAuthenticated() 允许认证过的,但排除remember-me的
hasAnyAuthority(String...) 具备某个权限
hasAnyRole(String ...) 具有某个Role
hasAuthority(String) 某有某个权限
hasRole(String) 具有某个Role
hasIpAddress(String) 具有某个IP
not() 对其他访问方法的结果取反
permitAll() 允许所有
rememberMe() 允许通过Remember-me的.
```
最后的default条件就是 `anyRequest().denyAll()`. 

`SpEL`: `Spring Expression Language`:
> access方法里头使用,可以代替其他方法,如:
access("hasRole('ROLE_READER' and hasIpAddress('192.168.1.2'))");

## Authentication与Authorization区别
`Authentication`: 鉴权,用户是谁.
`Authorization`: 授权,是否可以.
```java
// 获得用户是谁: 如果没有鉴权就是null
SecurityContextHolder.getContext().getAuthentication()
// 获得用户的权限:
SecurityContextHolder.getContext().getAuthentication().getAuthorities()
```



## 其他概念(自顶向下)

`SecurityContextHolder`:
    为使用者提供全局的SecurityContext//ThreadLocal变量,每个线程唯一.
`SecurityContext`:
    为了hold住Authentication
`Authentication`:// 鉴权
    主要负责两方面信息，一个是当前用户的详细信息(Principal、UserDetails)，一个是用户鉴权时需要的信息。
`Principal`: 安全主体
`Authorization`: 授权,访问控制.
`GrantedAuthority`:
    提供当前用户(UserDetails)所获得的系统范围内的授权。
`UserDetails`:
    提供了用户的详细信息，主要被用来构建Authentication。
`UserDetailsService`:   
    这个接口的实现主要是负责通过用户名查找并提供用户的详细信息(UserDetails)。
    
密码的比较:
由`AuthenticationManager`以及`AuthenticationProvider`负责.

## 鉴权流程
1. 用户名,密码 => `UsernamePasswordAuthenticationToken`对象;
//`Authentication`的一个实现.
2. => `AuthenticationManager`对它进行验证;
3. => 提取`UserDetails`,`GrantedAuthority` => `Authentication`对象
4. `Authentication`=> `SecurityContextHolder.getContext().setAuthentication`.



其中第一步在:
```
UsernamePasswordAuthenticationFilter类中:
UsernamePasswordAuthenticationToken authRequest = new UsernamePasswordAuthenticationToken(
				username, password);
```

# 第十章 模板方法设计模式
实现分为: 模板(template,固定部分)和回调(callback,可变部分)
Spring提供的模板包括:
```
jdbcTemplate
SimpleJdbcTemplate -- Spring3后已废弃
HibernateTemplate
JdoTemplate
JpaTemplate
```

# 第十一章 ORM
1. DSL
JPA repository的DSL(黑话):
```
readSpitterByFirstnameOrLastnameOrderByLastname();

动词: read,get,find,count. // 前三个等效
Spitter: 主题 // 还可以加Distinct
断言: 还可以有: isAfter,isNull,isLike,IgnoringCase等等.

```
只要按这个模式命名方法名,就会自动被实现.


2.`@Query`自定义查询
```java
@Query("select username,password,fullname from Reader")
    List<Reader> findAllGmailReaders();
```

3.混合自定义实现
上述情况都是声明一个`ReaderRepository`接口,
继承JPARepository<Reader,String>
(对象和对象主键类型),
Spring就会自动生成一个Impl,根据定义的方法名自动生成实现.(类似于用iBatis时,工具生成map)
如果有比较复杂的自定义SQL操作,可以创建一个`ReaderRepositoryImpl`,
和原来的名字相比增加`Impl`后缀,(这个是默认后缀,可以通过注解配置调整),
然后继承同一个接口(比如都继承/实现`ReaderSweeper`).
Spring就会混合自动生成的实现和自定义的实现.

//详见github.

# 第十二章 NoSQL
1. MongoDb 及其template
2. note4j 及其template
3. redis 及其template
依赖:

```groovy
compile "redis.clients:jedis:2.9.0"
compile 'org.springframework.boot:spring-boot-starter-data-redis'
```
yaml配置:

```yaml
spring:
  redis:
    host: localhost
    password: redis123
    port: 6379
```
配置:
```java
@Bean
    public RedisTemplate<String, Product> redisTemplate(RedisConnectionFactory cf) {
        RedisTemplate<String, Product> redis = new RedisTemplate<>();
        redis.setConnectionFactory(cf);
        return redis;
    }
```
最后一步如果用不着template的话,甚至可以不用配.还蛮简单的.

## 序列化相关:
Spring Data Redis提供的序列化器:
1. `RedisTemplate`会使用`JdkSerializationRedisSerializer`;
2. `StringRedisTemplate`默认会使用`StringRedis-Serializer`;
3. 可以在创建`RedisTemplate`的时候配置其他序列化器:
```java
 @Bean
    public RedisTemplate<String, Product> redisTemplate(RedisConnectionFactory cf) {
        RedisTemplate<String, Product> redis = new RedisTemplate<>();
        redis.setConnectionFactory(cf);
        redis.setStringSerializer(new StringRedisSerializer());
        redis.setValueSerializer(new Jackson2JsonRedisSerializer<Product>(Product.class));
        return redis;
    }
```

# 第十三章 缓存技术 (切面实现)
- EhCache 跳过.
1.声明缓存管理器
- RedisCacheManager :单个缓存管理器
```java
@Configuration
@EnableCaching
public class CachingConfig {
    @Bean
    public CacheManager cacheManager(RedisTemplate redisTemplate){
        return new RedisCacheManager(redisTemplate);
    }

}
```
- 多个缓存管理器:CompositeCacheManager
会迭代查找添加的各个缓存管理器.

2.使用缓存
4个相关注解: // 可用于类或方法级别.
```
@Cacheable : 先找之前缓存的值,没有则调用.
@CachePut :  只是放到缓存里,始终调用. // 一般用于save方法.
@CacheEvict : 清除n个缓存
@Caching : 分组
```

`@CachePut`:
```java
@CachePut(value="spittleCache", key="#result.id")
  Spittle save(Spittle spittle);
```

- 条件缓存:
```java
@Cacheable(value="spittleCache"
unless="#result.message.contains('Nocache')"
condition="#id >= 10"
Spittle findOne(Long id);
)
```

- 移除缓存:
```java
@CacheEvict("spittleCache")
void remove(long spittleid);
```

- 组合缓存操作:
```java
@Caching(   // 增加id查找,email查找,username查找的缓存
        put = {  
                @CachePut(value = "user", key = "#user.id"),  
                @CachePut(value = "user", key = "#user.username"),  
                @CachePut(value = "user", key = "#user.email")  
        }  
)  
public User save(User user) {...};
```

- 自定义注解:
```java
@Caching(  
        put = {  
                @CachePut(value = "user", key = "#user.id"),  
                @CachePut(value = "user", key = "#user.username"),  
                @CachePut(value = "user", key = "#user.email")  
        }  
)  
@Target({ElementType.METHOD, ElementType.TYPE})  
@Retention(RetentionPolicy.RUNTIME)  
@Inherited  
public @interface UserSaveCache {  
}  
```

# 第十四章 保护方法应用
1. 保护方法调用 // SecuredConfig
2. 使用表达式定义安全规则 //JSR250Config
3. 创建安全表达式计算器 //  ExpressionSecurityConfig
详见源码chapter_14.
配置上:
```java
@EnableGlobalMethodSecurity(securedEnabled=true)
public class SecuredConfig extends GlobalMethodSecurityConfiguration {

  @Override
  protected void configure(AuthenticationManagerBuilder auth) throws Exception {
    auth
    .inMemoryAuthentication()
      .withUser("user").password("password").roles("USER");
  }
```
相关注解:
```
@PreAuthorize 方法调用之前,用表达式限制调用
@PostAuthorize 方法调用之后,检查表达式,抛出异常
@PostFilter 允许方法调用,按表达式过滤结果
@PreFilter 允许方法调用,按表达式过滤输入值
```

service:
```java
@Secured({"ROLE_SPITTER", "ROLE_ADMIN"})
```

# 第十五章 远程调用
里头的RMI都是java的,不能跨语言.跳过.

# 第十六章 rest api
1. 内容协商 `Content negotiation`:选择一个能用的视图
2. 消息转化器 `Message conversion`:转换成客户端使用的形式

## 协商
一维协商:// 只有视图名
> Controller返回 -> 视图名 // 或void,原视图
-> DispatcherServlet -> 视图解析器. 
 
二维协商: // (视图名,内容类型)
- `ContentNegotiatingViewResolver`
> 内容协商的两个步骤:
确定请求的媒体类型;
找到适合媒体类型的最佳视图.

- 确定请求的媒体类型:
1.查看url的文件后缀名.
2.使用Accept首部类型
3.使用默认类型.

协商优点:
可以同时提供给人看的html和给客户端服务用的json. //重用controller
协商缺点:
客户端不能发送json或xml.
返回值是一个Model,也就是一个kv,比普通的多一层嵌套.

## 信息转换器
流程:
Accept头信息(application/json)
->处理方法返回的对象交给MappingJacksonHttpMessageConverter.
// 引入 Jackson JSON Processor库.

Spring提供的HTTP信息转化器包括:
```
AtomFeedHttpMessageConverter
BufferedImages
ByteArrayHttpMessageConverter
...
```

- `@ResponseBody`: 发送数据时使用转换器; 配合`produces`
```java
@RequestMapping(method=RequestMethod.GET, produces="application/json")
  public List<Spittle> spittles(
      @RequestParam(value="max", defaultValue=MAX_LONG_AS_STRING) long max,
      @RequestParam(value="count", defaultValue="20") int count) {
    return spittleRepository.findSpittles(max, count);
  }
```
- `@RequestBody`: 接受数据时使用转换器. 配合`consumes`
```java
@RequestMapping(method=RequestMethod.POST, consumes="application/json")
  @ResponseStatus(HttpStatus.CREATED)
  public ResponseEntity<Spittle> saveSpittle(@RequestBody Spittle spittle, UriComponentsBuilder ucb) {
    Spittle saved = spittleRepository.save(spittle);
    
    HttpHeaders headers = new HttpHeaders();
    URI locationUri = ucb.path("/spittles/")
        .path(String.valueOf(saved.getId()))
        .build()
        .toUri();
    headers.setLocation(locationUri);
    
    ResponseEntity<Spittle> responseEntity = new ResponseEntity<Spittle>(saved, headers, HttpStatus.CREATED);
    return responseEntity;
  }
```


- `@RestController`注解:
为所有方法增加@ResponseBody.方法的输入参数增加@RequestBody.
```java
@PathVariable long id
// 用路径中的{id}注入id; 
@RequestParam(value="max", defaultValue=MAX_LONG_AS_STRING) long max
// 用请求参数注入

@RequestMapping(method=RequestMethod.POST, consumes="application/json")
  @ResponseStatus(HttpStatus.CREATED)
 // 详见chapter_16 spittr-api-message-converters源码项目
```

# 第十七章 异步消息
- 简介
- JMS java消息服务 略过
- AMQP 高级消息队列服务 (rabbitMQ)
- POJO

## 简介
RMI是同步的,异步的只要投递完就返回.
1. 发送
- broker: 消息代理
- destination: 目的地

dest分为两种:
- topic 主题,发布/订阅模型
- queue 队列,点对点模型

## JMS
api规范.支持点对点和发布订阅.
//旧

## AMQP
高级MQ协议. 不但约束了api,还使不同实现之间可以合作.
加入了Exchange,Binding,解耦了队列,
可以灵活实现除了点对点\发布订阅以外的模型.

`Exchange`类型:
- Direct: 
> 判读消息和binding的key相等.
把消息路由到key相等的binding上.(binding的队列)

- Topic:  
> 判读消息key符合binding的通配符.
把消息路由到key匹配的binding上. 

- Headers: 
> 判断消息的headers与bingding参数匹配.

- Fanout:
> 无条件匹配. 广播到所有队列上.

默认会有一个没有名字的Direct Exchange, 所有队列都会绑定到这个Exchange上,并且routing key和队列名相同.

使用rabbitTemplate时,如果不指定exchange
,就会发送到默认的exchange,也就是上面说的默认Direct Exchange. 

// 其他内容写入<< rabbitmq in action笔记 >> 

# 第十八章 websocket
与http同级.
持久连接和前端通信(浏览器).用http的话,服务端无法向客户端发消息.
# 1. 直接使用websocket
前端: 使用ws://协议,或安全的wss://.
后端: 继承`AbstractWebSocketHandler`接口,引入依赖:
```
compile 'org.springframework.boot:spring-boot-starter-websocket'
```
为了防止浏览器不兼容的情况,使用sockjs:
```
registry.addHandler(marcoHandler(), "/marco").withSockJS();
```
相应的,前端也引入sockjs库.

# 2. 使用封装以后的STOMP
STOMP: Simple Text Oriented Messaging Protocol
简单文本的消息协议. 
主要是加了个代理,消息流如下:
```
两条消息,目的地分别为: /app/marco,/topic/polo
=>请求通道
=>/app: 发送到 AnnotationMethodMessageHandler => 代理通道
  /topic: SimpleBrokerMessageHandler
  /queue: SimpleBrokerMessageHandler
=>响应通道
```
如果要二进制的,还有别的协议MQTT.











# 第十九章 邮件
用qq邮箱代发即可,配置依赖以后,用springboot能自动获得相关bean,调用即可.不使用ssl的话需要配置:
```yaml
spring.mail.username: xxx@qq.com
spring.mail.password: password
spring.mail.port: 25 # SSL  465
spring.mail.protocol: smtp
spring.mail.host: smtp.exmail.qq.com # smtp.qq.com 
spring.mail.properties.mail.smtp.auth: true
```

# 第二十章 使用JMX
注册MBean到JMX中.
就试试Jconsole.

# 第二十一章 Springboot
起步依赖具体引入了什么:
```yaml
spring-boot-starter-actuator=>
 spring-boot-starter
,spring-boot-actuator
,spring-core

spring-boot-start-amqp=>
 spring-boot-start
,spring-boot-rabbit
,spring-core
,spring-tx

spring-boot-aop=>
 spring-boot-starter
,spring-boot-aop
,AspectJ Runtime
,AspectJ Weaver
,spring-core

spring-boot-starter-batch
 spring-boot-starter
,HSQLDB
,spring-jdbc
,spring-batch-core
,spring-core
... 太多了,以后有需要再查吧.
```