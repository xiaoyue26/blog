title: mybatis批量插入和更新
date: 2015-10-19 20:47:36
tags:
- mybatis
categories:
- orm
---


1. 批量插入:
`mysql`下批量插入的sql语句:
```sql
insert into user_info (uname,unumber) values 
('xiaoyue1','1'),
('xiaoyue2','2');
```
似乎很简单,
然而,经过复杂的条件判断和动态构造就变成了这样:
```xml
<insert id="insertMass" useGeneratedKeys="true" keyProperty="id">
		insert into user_info
		<trim prefix="(" suffixOverrides="," suffix=")">
			<if test="list[0].id != null">id,</if>
			<if test="list[0].uname!= null">uname,</if>
			<if test="list[0].unumber!= null">unumber,</if>
		</trim>
		values
		<foreach collection="list" item="item" index="index"
			separator=",">
			<trim prefix="(" suffixOverrides="," suffix=")">
				<if test="item.id != null">#{item.id},</if>
				<if test="item.uname!= null">#{item.uname},</if>
				<if test="item.unumber!= null">#{item.unumber},</if>
			</trim>
		</foreach>
</insert>
```

 - 用`useGeneratedKeys`和`keyProperty`接收自增主键返回的id,而且此处返回的是第一个插入的记录的id,而不是最后一条记录的id。注意调用代码传入的`map`中必须有`id`这一项以接收返回值。
 - 用`trim`标签消除末尾的逗号，同时在首尾加上括号。
 - 用`foreach`标签遍历传入的`list`, 传入的`map`中`key`为`list`的对象即为`userInfo`数组。
 - `list[0]`表示对象数组的第一项。

 

2. 对应的调用代码:
```java
@Override
	public void insertMass(List<UserInfo> users) {
		Map<String, Object> paraMap = new HashMap<>();
		paraMap.put("list", users);
		paraMap.put("id", 0);//note
		userInfoMapper.insertMass(paraMap);
		int beginId = (int) paraMap.get("id");
		for (UserInfo userInfo : users) {
			userInfo.setId(beginId);
			++beginId;
		}
	}
```

---

3. 批量更新:
本来是打算用`case when`语句写的,然而太过复杂没能写出来,可能得用`java`拼接好`sql`语句再传给`mybatis`；`replace into`虽然简练但可能改变其他不想改变的字段；最后折中写了`insert into..on duplicate key UPDATE...`。

```xml
<insert id="updateMass"> <!--on duplicate key, mysql特有语法 -->
		insert into user_info
		(
		<if test="list[0].unumber != null"> unumber,</if>
		uname,
		id<!-- id is necessary -->
		)
		values
		<foreach collection="list" item="item" index="index"
			separator=",">
			(
			<if test="item.unumber!= null"> #{item.unumber},</if>
			<if test="item.uname!= null">#{item.uname},</if>
			#{item.id}
			)
		</foreach>
		on duplicate key UPDATE
		<if test="list[0].unumber != null">unumber=values(unumber),</if>
		uname=values(uname) <!-- 至少更新一项 -->
</insert>
```
- 这次没使用`trim`标签，可读性好了一点点,可以和上面对比一下;
- 传入参数没设置类型,因为`mybatis`自己会判断, 其实类型是`map`。`map`中`list`项中存着要更新的对象数组。



