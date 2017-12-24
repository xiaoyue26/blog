---
title: springboot实战学习笔记
date: 2017-10-03 20:05:58
tags: 
- java 
- spring 
- springboot
categories:
- java
- spring
---


# 安装spring cli
略
# 创建项目
- 用网页创建:http://start.spring.io/
- 用Intellij创建.
> 如果是创建了gradle项目,一般需要加上插件:
apply plugin: 'idea'
执行:
gradle cleanIdea idea
才能正常使用.(idea生成的项目默认使用eclipse的插件,多么讽刺)

# JPA相关:
https://zhuanlan.zhihu.com/p/25000309

# 第二章:
- 依赖顺序:(可以与书写顺序相反,自顶向下或自底向上)
实体Book<=仓库(<Book,Long>)

## 注解
- 实体
```
@Entity 领域对象
@Id 主键
@GeneratedValue(strategy=GenerationType.AUTO) 自增id
其他就是getter,setter了.
```

- repository
```
interface xxx extends JpaRepository<Book, Long>
```

- controller
```
//@Controller 用于类
//@RequestMapping 用于类或方法
//@Autowired  可以对类成员变量、方法及构造函数进行标注，完成自动装配的工作。
// 最后返回视图名(String)

import java.util.List;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestMethod;

@Controller // 注册ctrl到spring context中,成为一个bean.
@RequestMapping("/readingList") // 匹配url. 
public class ReadingListController {

  private static final String reader = "craig";
  
	private ReadingListRepository readingListRepository;

	@Autowired  // 注入构造函数的输入参数
	public ReadingListController(ReadingListRepository readingListRepository) {
		this.readingListRepository = readingListRepository;
	}
	
	@RequestMapping(method=RequestMethod.GET)// 匹配GET
	public String readersBooks(Model model) {
		
		List<Book> readingList = readingListRepository.findByReader(reader);
		if (readingList != null) {
			model.addAttribute("books", readingList);
		}
		return "readingList";
	}
	
	@RequestMapping(method=RequestMethod.POST)// 匹配POST
	public String addToReadingList(Book book) {
		book.setReader(reader);
		readingListRepository.save(book);
		return "redirect:/readingList";
	}
	
}


```

## 条件化注解
```
@ConditionalOnBean 配置了某个特定Bean
@ConditionalOnMissingBean 没有配置特定的Bean
@ConditionalOnClass Classpath里有指定的类
@ConditionalOnMissingClass Classpath里缺少指定的类
@ConditionalOnExpression 
//给定的Spring Expression Language（SpEL）表达式计算结果为true
@ConditionalOnJava Java的版本匹配特定值或者一个范围值
@ConditionalOnJndi
//参数中给定的JNDI位置必须存在一个，如果没有给参数，则要有JNDI
InitialContext

@ConditionalOnProperty 指定的配置属性要有一个明确的值
@ConditionalOnResource Classpath里有指定的资源
@ConditionalOnWebApplication 这是一个Web应用程序
@ConditionalOnNotWebApplication 这不是一个Web应用程序
```

数据源相关spring源代码:// DataSourceAutoConfiguration
```
@Configuration
@ConditionalOnClass({ DataSource.class, EmbeddedDatabaseType.class })// classpath里必须有这两个类
@EnableConfigurationProperties(DataSourceProperties.class)
@Import({ Registrar.class, DataSourcePoolMetadataProvidersConfiguration.class })
public class DataSourceAutoConfiguration {
// do something
}
```

## springboot配置决策:
> 1. ClassPath里有H2时,创建一个嵌入式的H2数据库Bean.
//类型是javax.sql.DataSource,JPA实现(Hibernate)需要.
//org.springframework.orm里放的是Hibernate3,4,5代码

> 2. ClassPath里有Hibernate时,自动配置相关bean,包括:
Spring的实体管理相关,和JpaVendorAdaptor.

> 3. ClassPath里有Spring Data JPA时,自动配置为:
根据仓库接口创建仓库实现. (Repository相关代码)

> 4. ClassPath里有Thymeleaf时,将其配置为Spring MVC的视图,包括模板解析器,模板引擎及视图解析器.
视图会解析相对于classPath根目录的/templates目录里的模板. 

> 5. ClassPath里有Spring MVC时(web starter里有), 配置Spring DispatcherServlet并启用Spring MVC. 

> 6. 发现是SpringMVC web项目时,注册一个资源处理器,提供:
/static
/public
/resources
/META-INF/resources
中静态内容.

> 7. ClassPath里有Tomcat时(Web starter里有),启动嵌入式Tomcat. 

# 第三章 自定义配置
主要讲如何覆盖第二章中的默认配置.
1. 覆盖自动配置的bean
2. 用外置属性进行配置
3. 自定义错误页

- bean覆盖的原理:
springboot先加载应用级配置,然后使用@ConditionalOnMissingBean加载自动配置.
因此覆盖的第一种办法是重写一个完整的bean,比较繁琐.


- 用外置属性
微调一些属性. 
1. 运行时命令行参数中指定:
```
java -jar readinglist-0.0.1-SNAPSHOT.jar --spring.main.show-banner=false 
```
2. `application.properties`中指定:
```
spring.main.show-banner=false 
```
3. `为application.yml`中指定:
```
spring:
 main:
 show-banner: false 
```
4. 环境变量中指定:
```
export spring_main_show_banner=false 
```

优先级:
1. 命令行 -- 最优先
2. java:comp/env中的JNDI属性.
3. JVM系统属性
4. 操作系统环境变量
5. 随机生成的带random.*前缀的属性
6. 应用程序外的application.properties或yml文件
7. 应用程序内的application.properties或yml文件
8. 通过@PropertySource标注的属性源
9. 默认属性

位置优先级:
//application.properties或application.yml文件
1. 外置 运行目录的/config目录
2. 外置 运行目录
3. 内置 config包内
4. 内置 classPath根目录. 

##可以调整的属性例子:
1. 禁用模板缓存
```
spring:
 thymeleaf:
 cache: false 
```
同样适用于:
```
spring.freemarker.cache
spring.groovy.template.cache
spring.velocity.cache
```

2. 配置嵌入式服务器
```
server.port=8000
```
配置https服务的key:
```
keytool -keystore mykeys.jks -genkey -alias tomcat -keyalg RSA 
```
yml:
```
server:
 port: 8443
 ssl:
 key-store: file:///path/to/mykeys.jks 
 key-store-password: letmein
 key-password: letmein 
```

3. 日志配置
使用Log4j://默认使用logback
```
gradle:
configurations {
 all*.exclude group:'org.springframework.boot',
 module:'spring-boot-starter-logging'
} 

compile("org.springframework.boot:spring-boot-starter-log4j") 
//log4j2
compile("org.springframework.boot:spring-boot-starter-log4j2") 
```

单独的日志配置文件:
```
logging:
 config:
 classpath:logging-config.xml 
```
日志级别:
```
logging:
 path: /var/logs/
 file: BookWorm.log
 level:
 root: WARN
 org:
 springframework:
 security: DEBUG 
```

4. 配置数据源
```
spring:
 datasource:
 url: jdbc:mysql://localhost/readinglist
 username: dbuser
 password: dbpass 
```
数据库连接池寻找顺序://ClassPath
1. tomcat
2. HikariCP
3. Commons DBCP
4. Commons DBCP2

打开debug看日志的话,`dataSourceInitializer`数据源初始化的时候会自动执行resources目录下的`data.sql`文件.


>@ManyToOne等数据库相关注解:
https://www.cnblogs.com/yxjdragon/p/6008603.html
1. 一个A对应多个B:
方法1:A里设置主键;B里使用@ManyToOne设置关联;
方法2:A里使用@OneToMany,Set<User>;B里设置主键. 


## 第三种配置方式: 应用程序Bean的配置外置
详见:
https://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/#boot-features-profiles

真的是博大精深,千变万化.
代码示例中的注入过程:
```
网页模板中${amazonID}
<= model.addAttribute("amazonID", amazonConfig.getAssociateId());
<= amazonConfig是AmazonProperties的一个对象:
AmazonProperties:
@Component
@ConfigurationProperties("amazon")
public class AmazonProperties {

  private String associateId;
  
  public void setAssociateId(String associateId) {
    this.associateId = associateId;
  }
  
  public String getAssociateId() {
    return associateId;
  }
  
}

<= Controller的构造函数中注入:
@ConfigurationProperties("amazon")
public class ReadingListController {
@Autowired// 自动注入方法的参数
	public ReadingListController(ReadingListRepository readingListRepository,
								 AmazonProperties amazonConfig) {
		this.readingListRepository = readingListRepository;
		this.amazonConfig = amazonConfig;
	}
}

<= 最根源: application.yml文件:
amazon:
  associate_id: habuma-20

```



# 第四章: 单元测试
1. 用Mock进行测试
2. 用Mock,带着登录用户信息
3. 用模拟器测试(Selenium)
4. 用模拟器测试(Selenium),带着cookie(这个没写)

# 第五章: Groovy与Spring Boot CLI 
似乎是凑字数的一章. 大致跑了一下代码,浏览了一下.
# 第六章: 
感觉是私货,大致扫了一下内容,代码都没跑.

先配数据源:
http://www.jianshu.com/p/34730e595a8c


# 注解笔记
阅读过程遇到的注解越来越多,有必要总结一下:

## 1. 声明Bean的注解: 其实可以都用@Component
- `@Service`用于标注业务层组件
- `@Controller`用于标注控制层组件（如struts中的action）
- `@Repository`用于标注数据访问组件，即DAO组件
- `@Component`泛指组件，当组件不好归类的时候，我们可以使用这个注解进行标注。
- `@Bean`声明一个bean,还可以定义的属性:name,initMethod,destroyMethod,autowire(装配策略:`Autowire.BY_NAME`，`Autowire.BY_TYPE`，`Autowire.NO`)
- `@Primary`, 修饰这个Bean应优先被使用.// @Qualifier则是在使用的时候显示指定使用哪个.

还有一个有用的注解:
- `@Order`: 声明装配的顺序.经过测试,加载的顺序还是字母序,不受影响. 如果装配到List<IRank>容器里,装配的顺序由@Order决定.以此类推,同级别的切面实现,执行顺序也是按@Order. 

```
// 使用样例:
@Order(100)
public abstract class WebSecurityConfigurerAdapter implements
// 源码:
public @interface Order {
    int value() default 2147483647;
}
```


默认Bean的名字: 
    > 类名(头字母小写)

更改Bean的名字:
    > @Service(“beanName”)
    
默认Bean是单例,要改变:
    > @Scope(“prototype”)

`bean`的生命周期   
> 1. singleton: 默认是单例,与容器共存亡,基本就是始终存活;
2. prototype: 每次都是新的,与单例相反,调用者自己负责销毁;
-- 后三种是新增的,专用于web应用的
3. request: 每个HTTP请求一个,是prototype的一种扩展;
4. session: 每个Session一个,但咱公司一般不用session,用cookie;
5. global session: 基于porlet的web应用程序中才有意义.

    
增加初始化方法:
```
@PostConstruct public void init() { }
```
增加析构方法:
```
@PreDestroy public void destory() {}
```
    
## 2. 使用Bean的注解:
-  `@Autowired` 不需要写getter和setter
-  `@Qualifier` 如果同一个接口有多个实现类的话,需要指定具体注入哪一个:
    > @Qualifier("chinese")       
-  `@Resource` 约等于`@Autowired`,区别是默认按name进行注入.也可以选择类型.

`@Autowired`:

> 构造函数注入和字段注入的区别:(Spring场景下)
1. 构造函数注入时,spring能检测循环依赖并报错;而字段注入的时候,spring检测出来以后会想办法解决,不会报错.
2. 构造函数注入时,能控制注入顺序.
3. 字段注入时,spring能按需注入,写起来简单;


- 字段注入的缺点:
1. 不能创建不可变对象;
2. 与DI容器紧耦合,不能在类外使用;
3. 除了反射,类不能被实例化(例如在单元测试中).//只能进行集成测试
4. 从外部(接口或构造函数)无法看出依赖,依赖情况隐藏在实现中.
5. 如果有循环依赖,并不会报错,隐藏了可能的错误.
// 如果一个类依赖太多,很可能是违反了设计原则.





原则:
1. 必须项\不可变项: 使用构造函数注入;
2. 可选项\可变项: 使用字段注入;
3. 大部分情况下避免字段注入. 尽量使用构造函数注入.

## 3.配置相关的注解:
- `@EnableAutoConfiguration`
> 启动自动配置,就是根据classpath中的jar,自动配置. 

- `@SpringBootConfiguration`
> 继承自`@Configuration`, 标记当前类为配置类. 
实例代码:
```

@Configuration
@EnableAutoConfiguration
@ComponentScan
public class SecurityConfig extends WebSecurityConfigurerAdapter {
    @Autowired
    private ReaderRepository readerRepository;
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http
                .authorizeRequests()
                .antMatchers("/").access("hasRole('READER')")
                .antMatchers("/**").permitAll()
                .and()
                .formLogin()
                .loginPage("/login")
                .failureUrl("/login?error=true");
    }
    @Override
    protected void configure(
            AuthenticationManagerBuilder auth) throws Exception {
        auth.userDetailsService(userDetailsService());
    }
    @Bean
    public UserDetailsService userDetailsService() {
        return new UserDetailsService() {
            @Override
            public UserDetails loadUserByUsername(String username)
                    throws UsernameNotFoundException {
                UserDetails userDetails = readerRepository.findOne(username);
                if (userDetails != null) {
                    return userDetails;
                }
                throw new UsernameNotFoundException("User '" + username + "' not found.");
            }
        };
    }
}
```
`Spring`在解析到这种类时，将识别出标注`@Bean`的所有方法执行，
并将方法的返回值 (UserDetailsService)注册到IoC容器中。
默认情况下，Bean 的名字为方法名。

- `@ComponentScan`
> 扫描当前包及其子包下声明为Bean的类.(包括@Configure,@Contrller等等).
纳入Spring容器.
可以通过`@SpringBootApplication(scanBasePackages = "com.xxx.yyy")`,更改扫描的范围.//子包相关:default包并不是其他包的父包.


## 4.复合注解:

- @SpringBootApplication 包含:
> @SpringBootConfiguration
> @EnableAutoConfiguration
> @ComponentScan 等

- 网上教材:
https://doc.yonyoucloud.com/doc/Spring-Boot-Reference-Guide/IX.%20%E2%80%98How-to%E2%80%99%20guides/71.3.%20Enable%20HTTPS%20when%20running%20behind%20a%20proxy%20server.html

# 第七章 深入Actuator
- Actuator Web端点
- 调整Actuator
- 通过shell连入运行中的应用程序
- 保护Actuator 

## 7.1 Actuator的Web端点
HTTP方法 路径  描述
```
GET /autoconfig 自动配置报告.记录各自动配置条件通过与否.
GET /configprops 描述配置属性(含默认值)如何注入Bean
GET /beans   描述上下文中所有Bean及关系.
GET /dump    获取线程活动的快照
GET /env     获取全部环境属性
GET /env/{name} 根据名称获取特定的环境属性
GET /healthy    报告app的健康指标(由实现类提供)
GET /info       获取app的定制信息(由info开头的属性提供)
GET /mappings   描述全部的URI路径及其控制器(含Actuator端点)的映射关系
GET /matrics    报告各种应用程序度量信息,比如内存用量和HTTP请求计数
GET /trace      提供基本的HTTP请求跟踪信息(时间戳,HTTP头)
POST /shutdown  关闭app,要求endpoints.shutdown.enabled设置为true
```


## 7.2 查看配置明细
> 知道SpringBoot具体怎么装配的组件.

配置yml :
```
endpoints.sensitive: true
management.context-path: /admin
management.security.enabled: true
```

### /beans 可读
用户权限上增加角色`new SimpleGrantedAuthority("ROLE_ACTUATOR")`;
然后访问:
http://localhost:8000/admin/beans
获取所有Beans的信息.包括依赖,以及具体是使用JDK动态代理的还是CGLIB代理的.


>基于JDK动态代理 ，可以将@Transactional放置在接口和具体类上。
基于CGLIB类代理，只能将@Transactional放置在具体类上。

### /autoconfig 难读
- 条件化配置(自动配置)信息,`positiveMatches`以及`negativeMatches`,比较难看懂,真正有需要的时候进去搜索即可:
http://localhost:8000/admin/autoconfig

### /env 可读,易读
- 环境变量信息
> 包括启用的是哪个profiles,用的哪个log配置,各种运行时变量.
其中数据库连接是明文,数据库密码是星号代替.因此密码不能写到数据库连接里.

### /configprops 可读
- 配置属性报告
可以看到有哪些属性被注入了Bean,可以从中学到有哪些属性可以在yml中设置.

### /mappings 可读
- 端点和控制器的映射.
可以看到所有url都映射到了哪个controller,可以从中学到有哪些url可以使用.

### /metrics 可读
- 运行时统计. 
包括线程,内存,数据库连接活跃等信息.

### /trace 可读
- 报告所有Web请求的详细信息，包括请求方法、路径、时间戳以及请求和响应的头信息。实际能显示最近100个请求的信息,似乎对于测试应用有用,线上应用请求量一大,100条里应该就不会有想要的日志了.

### /dump 难读
- 当前活跃线程快照.
太长了.不过每个线程的信息很短.

### /health 可读
- 当前程序,磁盘,数据库是否活着. 

### /shutdown POST
发送关闭命令:
```
curl -X POST http://localhost:8000/admin/shutdown 
```
如果要身份验证,命令就比较不好搞了,需要python脚本或者别的代码辅助.

### /info 可读
显示在yml里配的info属性的值.比如联系方式.

## 其他
接下来书里介绍了用shell完成和REST api一样的功能,感觉有点难用.

## JMX监控
```
jconsole [进程号]
// eg. jconsole 61611
```
除了进程\线程运行的信息,在MBean面板还有`org.springframework.boot`目录,下面有REST api中的属性,可以通过data属性的值得到相同的值.
值得注意的是,与REST api不同的是,一旦启用了shutdown,即使不登录用户,也可以通过Jconsole关闭app. 

## 其他
接下来介绍了如何自定义端点url,以及加入自定义内容.
还可以把/trace接口的内容存到mongodb. 
细节略过不提.

## 第八章 部署
讲了下如何部署的细节,网上有别的教程.各种对照着做才行.

## 附录
包括开发工具,远程调试. 如果有需要可以看看.
剩下的是起步依赖的一些详细信息.

这本书还蛮短的.

# CommandLineRunner接口
CommandLineRunner、ApplicationRunner 接口是在容器启动成功后的最后一步回调（类似开机自启动）。
可以在集成测试的时候使用.