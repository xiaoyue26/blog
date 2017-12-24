title: springmvc笔记
date: 2015-10-02 17:49:41
tags:
- springmvc
categories:
- spring

---


1. 调用顺序：
(1)以`web.xml`为入口，(`context-param`里各种环境的配置)编码由`filter-mapping`拦截，`servlet-mappin`g匹配`servlet`所服务的url模式,例如当此`servlet`加载配置`classpath:spring-mvc.xml`之后：
(2)扫描`base-package`下的`controller`，`controller`中`@RequestMapping`匹配url模式，调用`service`，调用`service.impl`，调用`Mapper(DAO)`，调用`mapping`包中`mapper.xml`中的`sql`语句。
所以编写顺序：
```
mapper.xml
Mapper(DAO)
service
service.impl
controller
```
调用顺序：
```
controller
service.impl
Mapper(DAO)
mapper.xml
```
2.需要检查注解的地方：
```
Controller：
@Controller  

@RequestMapping("/user")  类

@Autowired (service)

@RequestMapping("/showInfo/{userId}")  方法
#两层递进的request mapping，url可以不止两层，因为方法前的url可以很多斜杠，类前的不知道。

service.impl:

@Service("userService")

@Autowired mapper
```
方法前注解记录：
```
@ResponseBody
标注后，返回String对象的结果为response内容体，不标注的话，作为dispatcher url使用。
@PathVariable 
允许将请求路径的一部分当做方法的传入形参使用
```
方法的返回值：
```
void: 
	此时逻辑视图名由请求处理方法对应的URL确定，如请求welcome,则返回welcome.jsp
String:
	此时逻辑视图名为返回的字符，如return "/user/showInfo";则返回/user/showInfo.jsp。  
ModelMap：
	跟void一样。
```
```
spring-context:
1.将Cinema.java的头部标注为@Component说明该类交由Spring托管
2.Cinema.java中的属性MoviceService标注为@Autowired，则Spring在初始化Cinema类时会从Application Context中找到类型为MovieService的Bean，并赋值给Cinema
3.在Application.java中我们声明了一个类型为MovieService的Bean。并且标注Application.java为@Configuration,这是告诉Spring在Application.java中定义了一个或多个@Bean方法，让Spring容器可以在运行时生成这些Bean。
4.@ComponentScan则会让Spring容器自动扫描当前package下的标有@Component的class，这些class都将由Spring托管。
```
此外还有未开启的注解，测试项目中未能涉及。
运行步骤：
```
1. maven build, goal: clean compile package
2. run on server
3. http://localhost:8080/sunday/user/showInfos.htmls
4. http://localhost:8080/sunday/user/showInfo/1.htmls 显示id为1的数据
   http://localhost:8080/sunday/user/showInfo/4.htmls 显示id为4的数据
```
3.Sql_map中带有的两个函数是：
```
updateByPrimaryKeySelective
updateByPrimaryKey
```
前者只是更新新的model中不为空的字段。
后者则会将为空的字段在数据库中置为NULL。

http://blog.csdn.net/hao947_hao947/article/details/34594695?utm_source=tuicool
```
resultMap ：  
id：resultMap的唯一标识
type：映射实体类的数据类型
property：实体类里的属性名
column:库表的字段名
<!--id:当前sql的唯一标识  
         parameterType：输入参数的数据类型   
         resultType：返回值的数据类型   
         #{}:用来接受参数的，如果是传递一个参数#{id}内容任意，如果是多个参数就有一定的规则,采用的是预编译的形式
select * from person p where p.id = ? ，安全性很高 -->

<!-- sql语句返回值类型使用resultMap -->  
<select id="selectPersonById" parameterType="java.lang.Integer"  
        resultMap="BaseResultMap">  
        select * from person p where p.person_id = #{id}  
</select>  
<!-- resultMap:适合使用返回值是自定义实体类的情况   
    resultType：适合使用返回值的数据类型是非自定义的，即jdk的提供的类型 -->  
<select id="selectPersonCount" resultType="java.lang.Integer">  
        select count(*) from  
        person  
</select>  

<select id="selectPersonByIdWithMap" parameterType="java.lang.Integer"  
        resultType="java.util.Map">  
        select * from person p where p.person_id= #{id}  
</select>  
```
http://www.open-open.com/lib/view/open1325862592062.html
`ofType`也是表示返回类型，这里的返回类型是集合内部的类型，之所以用ofType而不是用type是MyBatis内部为了和关联association进行区别。
```
<resultMap type="Blog" id="BlogResult">

	<id column="id" property="id"/>
	<collection property="comments" select="selectCommentsByBlog" column="id" ofType="Comment"/>
</resultMap>

<resultMap type="Comment" id="CommentResult">
	<association property="blog" javaType="Blog" column="blog" select="selectBlog"/>

</resultMap>
```
	注意collection和association，ofType和javaType的不同。此处column="blog"中的blog为外键名。

	第一次猜测#{title}中引用的应该是传入参数对象的实体类中的属性名。(应该不是数据库中的列名。)
	比如传入参数类型是Blog,#{blogid}就应该是Blog类的blogid属性。 

	在看http://zhuyuehua.iteye.com/blog/1721715之后：
```
<select id="selectRole" parameterType="java.lang.Long" resultType="Role" >     
    select * from Role where id =#{id}     
</select>   
	selectRole的SQL输入参数可以随便给名称，只要是输入参数与压入进去的值类型相同就行了，可以写成：
	select * from Role where id = #{sfffs}   
	mybatis最终会执行：select * from role where id =resultSet.getLong("role_id");  
	其中传入参数的语句可能是：
<association property="role" column="role_id" javaType="Role" select="selectRole"/>  
```
存疑。
	继续看http://zhuyuehua.iteye.com/blog/1721715。


