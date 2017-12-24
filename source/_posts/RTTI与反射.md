title: RTTI与反射
date: 2015-10-18 20:53:45
tags:
- java
categories:
- java
---


>- `RTTI`: 
    运行期类型鉴定。在编译期通过载入和检查`.class`文件,从而鉴定类型。
>- `反射`: 
    运行期类信息。应用场景:反序列化,远程调用(RM)，访问私用变量和方法。编译期`.class`文件不可用,只能在运行期获得`.class`文件，在运行期载入和检查`.class`文件。

1.`RTTI`:
```java
//RTTI的例子:
if(boolean.class==Boolean.TYPE)
    System.out.println("boolean true");
Gun gun=new Gun();
if//(Gun.class.isInstance(gun))//方式1;
(Class.forName("test1.RTTI$Gun").isInstance(gun ))//方式2反射。
    System.out.println("gun is instance of  Gun");
    //此处Gun是test1包下RTTI类中的一个内部类。
```
2.反射:
http://blog.csdn.net/stevenhu_223/article/details/9286121
```java
Class<?> class1 = Class.forName("test1.RTTI$Gun");
Method[] methods = class1.getMethods();
for (Method method : methods)
	out.println(method);
out.println("constructos:--");
Constructor[] constructors=class1.getConstructors();
for (Constructor method : constructors)
	out.println(method);
Method me1=class1.getMethod("getdata", int.class);
Type returnType=me1.getReturnType();
Object output=me1.invoke(class1.newInstance(),3);
out.println(output);

```
>`getFields`方法获取所有`public`属性；
>`getDeclaredFields`方法获取所有属性,包括`private`。
>`getMethods`方法获取所有`public`方法;
>`getDeclaredMethods`方法获取所有方法,包括`private`。

可通过`getDeclaredMethods`获取私有方法；
然后`nameMethod.setAccessible(true);  `
然后就可以执行`private`方法了。

此外，获取方法时，除了指定方法名，还要指定参数类型列表，多个参数时可这样写:
```java
Class<?> clazz=Gun.class;
Method allValuesMethod = clazz.getDeclaredMethod("setAllValues", new Class[]{String.class, int.class, float.class});  
```
