title: mybatis处理mysql自增主键的三种写法
date: 2015-10-04 08:18:16
tags:
- mysql
- mybatis
categories:
- orm

---


1. 数据库的自增主键id(int(11)),查看`auto_increment`,记录的是下一次插入时的主键值。(=最大的id+1)
```sql
SHOW TABLE STATUS;
#或:
SELECT AUTO_INCREMENT FROM  information_schema.tables WHERE table_name='t_account_0'
```
2. 测试`select @@identity`命令：
```
INSERT into user_info  (uname, unumber) VALUES ('test','test');
select @@identity  as 'curMaxId';
```
注意到第一条命令并没有插入`id`值，以便让数据库插入自增`id`值。
两条命令同一线程中顺序执行后，第二条命令获取到第一条命令插入的`id`值。
例如，若`select @@identity`结果为`106`，则`auto_increment`此时为`107`，而`INSERT`的记录`id`值为`106`.

3. 第一种mybatis支持自增主键的写法，使用`useGeneratedKeys`和`keyProperty`属性设置：
```
<sql id="UserInfoSet">
		<set>
			<if test="id != null">id=#{id},</if>
			<if test="uname!= null">uname=#{uname},</if>
			<if test="unumber != null">unumber=#{unumber},</if>
		</set>
	</sql>
<insert id="insert" parameterType="UserInfo"
		 useGeneratedKeys="true" keyProperty="id">
		insert user_info
		<include refid="UserInfoSet" />
</insert>
```
如果表中主键已经设定为`auto_increment`，则插入后会自动把生成的`id`值返回给插入的对象。
```java
UserInfo userInfo = new UserInfo();
userInfo.setUname("xiaoming"); 
userInfo.setUnumber("x07");
//a.插入前userinfo的id为null
result = userService.insert(userInfo);		
//b.插入后userinfo的id为数据库生成的id值(例如123).
```

4. 第二种使用`selectKey`，限定于`mysql`数据库的写法,其他数据库如`oracle`为`before`,各有不同:
```
<insert id="insert" parameterType="UserInfo">
	<selectKey resultType="java.lang.Integer" keyProperty="id" order="AFTER">
	SELECT @@IDENTITY
    </selectKey>
insert user_info <include refid="UserInfoSet" />
</insert>
```
此时表现和上一种一样;
若把`SELECT @@IDENTITY`改为 `select 20 from dual` , 则数据库中虽依然插入了正常递增的id值(如`123`),
但`b`处插入后的对象中id值为`20`. 可见`selectKey`标签实际执行是在`insert`执行后，返回`obj`对象前,将其中的`id`值改为`20`.并不影响`insert`执行过程。
综上所示，这两种方法其实都不影响`insert`,只是执行后获取一次`id`值。`insert`过程中生成`id`值是由数据库支持的。
所以如果数据库中没有设定主键的自增属性，这两种都是无能为力的,此时就要更改SelectKey的用法:

5. 第三种，使用`select max(id)+1`：
```
<insert id="insert" parameterType="test.model.UserInfo" >
	<selectKey resultType="java.lang.Integer" keyProperty="id" order="BEFORE">
	select max(id)+1 from user_info
	</selectKey>
	insert user_info <include refid="UserInfoSet" />
</insert>
```
其中`select max(id)+1 from user_info`语句是选出表中最大id值+1,并赋予传入的参数`userInfo`对象,然后再执行`insert`.
注意到`order`此时等于`before`. 应当警惕的是多用户交替插入时,这种方法的安全性,涉及到事务比较麻烦,所以最好还是直接设计数据库时直接设定主键的递增属性。

其他感想: 
    `mybatis`的`choose when`好比`switch case`;
`otherwise`就是`default`。`foreach`就是传入一个`list`或`array`给`mybatis`，然后生成很长的`sql`。
    学到这里大概觉得`mybatis`生成`sql`，就跟`jsp`生成`html`一样，就是输出，就是翻译，编译原理无处不在，重复造轮子无处不在。
暂时不适用`trim`标签了，太geek了，而且居然只有replace功能，可读性差。

