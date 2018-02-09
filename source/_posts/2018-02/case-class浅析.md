---
title: case class浅析
date: 2018-02-08 20:07:06
tags: scala
categories:
- spark
---

# 结论
`case class`又称样例类,可用于DTO(序列化特性),模式匹配(unapply特性).
作为一种语法糖,在定义之后会隐式获得很多经典的方法.

## 1. case class与case object
无参数时,使用`case object`;
有参数时,使用`case class`.
`case object`比参数为`()`的`case class`少`apply`和`unapply`方法.

## 2. 类参数
类参数默认作为`private final`的类成员.
同时自动生成一个同名`public`方法以获取它的值.

## 3. 伴生对象
- `apply`方法和`unapply`方法:
`case class`会自动生成一个伴生对象(`object`),作为当前作用域的一个单例,同时在里面实现了`apply`方法,可以使用`apply`方法或者括号函数`()`创建对象.(也可以依旧使用`new`关键字)
由于有`unapply`方法实现,因此可以用于模式匹配.

## 4. 序列化特性








# 详情

## 1. 用IDE查看class文件
假如写这么一个scala文件:
```
package learn
case class TestCaseClass()
```


编译后生成的class文件会有两个:
```
TestCaseClass$.class
TestCaseClass.class
```
用IDE打开`TestCaseClass.class`是:
```
package learn
case class TestCaseClass() extends scala.AnyRef with scala.Product with scala.Serializable {
}
```
可见定义`case class`后,自动会继承`AnyRef`,加上`Product`和`Serializable`的`trait`.
另一个内部类文件`TestCaseClass$.class`虽然用ide打开为空,但看文件大小1180,可见并不是空文件.

## 2. 反编译
使用`javap -p -c -s TestCaseClass.class`查看,过滤掉看不懂的部分和常量池里的各种符号引用,大致能发现如下内容:
- 类签名
```
public class learn.TestCaseClass implements scala.Product,scala.Serializable
```
可见本质上是实现Product和Serializable的接口.(加上默认实现)

- 方法:
```
public static boolean unapply(learn.TestCaseClass);
public static learn.TestCaseClass apply();
public learn.TestCaseClass copy();
public java.lang.String productPrefix();
public int productArity();
public java.lang.Object productElement(int);
public scala.collection.Iterator<java.lang.Object> productIterator();
public boolean canEqual(java.lang.Object);
public int hashCode();
public java.lang.String toString();
public boolean equals(java.lang.Object);
public learn.TestCaseClass();
```
注意到`unapply`和`apply`方法是`static`的,说明会自动生成一个伴生对象,并添加这两个方法.(`static`方法在`object`里)

### 反编译内部类:
- 类签名:
```
public final class learn.TestCaseClass$ extends scala.runtime.AbstractFunction0<learn.TestCaseClass> implements scala.Serializable
```
可见`case class`其实也是作为一个`Function`存在的,如果把源码改动一下加上参数:
```
package learn
case class TestCaseClass(xyz:String)

```

则这里的类签名变为:
```
public final class learn.TestCaseClass$ extends scala.runtime.AbstractFunction1<java.lang.String, learn.TestCaseClass> implements scala.Serializable
```
此外由于类参数没有加限定,默认变成了`val`,因此类参数是`private final`的:
```
private final java.lang.String xyz
```

其中`AbstractFuction`是一个抽象类,源码为:
```
abstract class AbstractFunction0[@specialized(Specializable.Primitives) +R] extends Function0[R] {}
```


- 方法签名:
```
public static {};// 3: invokespecial #15  // Method "<init>":()V
public final java.lang.String toString();
public learn.TestCaseClass apply();
public boolean unapply(learn.TestCaseClass);
public java.lang.Object apply();
```

