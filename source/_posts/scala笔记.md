---
title: scala笔记
date: 2017-05-17 20:02:38
tags: scala
categories:
- spark
---

http://www.runoob.com/scala/scala-data-types.html
scala api:
http://www.scala-lang.org/api/current/scala/Nothing.html
http://blog.csdn.net/bluishglc/article/details/55668192

# 运算优先级:
> 在Scala里所有以“:”结尾的运算符是右关联的，其他的运算符都是左关联的

# 数据类型
```scala
// Byte,Int,Short,Long等略.
Unit : 表示void. 表示无值，和其他语言中void等同。用作不返回任何结果的方法的结果类型。Unit只有一个实例值，写成()。

Null : null 或空引用,Null类是null引用对象的类型，它是每个引用类（继承自AnyRef的类）的子类。Null不兼容值类型

Nothing : Nothing类型在Scala的类层级的最低端；它是任何其他类型的子类型。

Any : Any是所有其他类的超类
// Any有两个子类：AnyVal和AnyRef

AnyRef	: AnyRef类是Scala里所有引用类(reference class)的基类

```
Nothing和Unit的区别:
> 1. Unit有一个值 ()
Nothing没有值.
2. 以Unit为返回值的话,方法能正常结束;
以Nothing为返回值的话,方法只能以异常退出.



http://blog.csdn.net/bluejoe2000/article/details/30465175

# 变量与常量:
- val
> value,值. 定义时立即求值. (饿汉求值). 只求一次.

- var
> variable,变量.  定义时立即求值. 可改变赋值. 只求一次.

- def
> define. 每次使用时才求值.(惰性求值). 求N次

- lazy val
> 懒求值. 第一次使用时求值,但只求一次.

- 退出cli
> :q 或 sys.exit

- 查看版本:
> scala --version

# 访问修饰符
> private: Scala 中的 private 限定符，比 Java 更严格，在嵌套类情况下，外层类甚至不能访问被嵌套类的私有成员。
```
class Outer{
    class Inner{
    private def f(){println("f")}
    class InnerMost{
        f() // 正确
        }
    }
    (new Inner).f() //错误
}
```

>protected:在 scala 中，对保护（Protected）成员的访问比 java 更严格一些。因为它只允许保护成员在定义了该成员的的类的子类中被访问。而在java中，用protected关键字修饰的成员，除了定义了该成员的类的子类可以访问，同一个包里的其他类也可以进行访问。
```
package p{
class Super{
    protected def f() {println("f")}
    }
	class Sub extends Super{
	    f() // 正确
	}
	class Other{
		(new Super).f() //错误
	}
}
```

>public: 任何地方都可以被访问

> private[x],protected[x]:这里的x指代某个所属的包、类或单例对象。如果写成private[x],读作"这个成员除了对[…]中的类或[…]中的包中的类及它们的伴生对象可见外，对其它所有类都是private。
这种技巧在横跨了若干包的大型项目中非常有用，它允许你定义一些在你项目的若干子包中可见但对于项目外部的客户却始终不可见的东西。

```
package bobsrocckets{
    package navigation{
        private[bobsrockets] class Navigator{
         protected[navigation] def useStarChart(){}
         class LegOfJourney{
             private[Navigator] val distance = 100
             }
            private[this] var speed = 200
            }
        }
        package launch{
        import navigation._
        object Vehicle{
        private[launch] val guide = new Navigator
        }
    }
}

```
上述例子中，类Navigator被标记为private[bobsrockets]就是说这个类对包含在bobsrockets包里的所有的类和对象可见。
比如说，从Vehicle对象里对Navigator的访问是被允许的，因为对象Vehicle包含在包launch中，而launch包在bobsrockets中，相反，所有在包bobsrockets之外的代码都不能访问类Navigator。


```
class Student{  
  var age=20     //底层编译器会自动为age添加get和set的公有方法 
  private[this] var gender="male" 
  //private[this] 只有该类的this可以使用  
  private var name="clow"
  //声明了private,底层编译器会自动为私有的name添加get和set的私有方法  
  //但是可以自己定义属性方法  
  def getName=this.name  
  def setName(value:String){this.name=value}  
}  
//构造器的使用  
class Teacher {  
  var age: Int = _  
  var name: String = _  //可以预留  
  
//重载的构造器 和public Teacher(){}类似  
def this(age: Int, name: String){  
    this() //必须得调用一次主构造器  
    this.age=age  
    this.name=name  
}  
}  
```
# 下划线的用法
```
// 1、作为“通配符”，类似Java中的*。如
import scala.math._


// 2、:_*作为一个整体，告诉编译器你希望将某个参数当作参数序列处理
val s = sum(1 to 5:_*)



3、//指代一个集合中的每个元素。例如我们要在一个Array a中筛出偶数，并乘以2，可以用以下办法：
a.filter(_%2==0).map(2*_)
//又如要对缓冲数组ArrayBuffer b排序，可以这样：
val after = a.sortWith(_.compareTo(_) > 0)
// 默认顺序: val aSorted = a.sorted

4、在元组中，可以用方法_1, _2, _3访问组员。如a._2。其中句点可以用空格替代。


5、使用模式匹配可以用来获取元组的组员，
例如
val (first, second, third) = t
但如果不是所有的部件都需要，那么可以在不需要的部件位置上使用_。比如上一例中
val (first, second, _) = t
val (first, _) = (1, 2)

6、还有一点，下划线_代表的是某一类型的默认值。对于Int来说，它是0。对于Double来说，它是0.0对于引用类型，它是null.

```
https://my.oschina.net/leejun2005/blog/405305

# scala的函数和方法:
> Scala 有函数和方法，二者在语义上的区别很小。Scala 方法是类的一部分，而函数是一个对象可以赋值给一个变量。
```
函数也类似于一个变量,类似函数指针的概念.
stackoverflow上推荐的代码风格是:
1. 加上 = 号;
2. 加上函数的返回值类型.

“=”并不只是用来分割函数签名和函数体的，它的另一个作用是告诉编译器是否对函数的返回值进行类型推断！如果省去=,则认为函数是没有返回值的！

闭包:
闭包是一个函数，返回值依赖于声明在函数外部的一个或多个变量。
闭包所引用的外部变量,会被trace,在闭包context内都不会被gc释放.
// 我看来闭包就是一个侧漏的函数,一点也不封闭. 作用是用于一些延迟执行的函数. 比如传递给spark的rdd. 
```

# 传参:
```
//传值调用（call-by-value）：先计算参数表达式的值，再应用到函数内部；

//传名调用（call-by-name）：将未计算的参数表达式直接应用到函数内部
// 类似于宏替换
def delayed( t: => Long ) = {
    // ...
   }
```

# List符号函数
> 优先使用 `++` 而不是 `:::` ;
> 优先使用 `+:` 而不是 `::`;
> 冒号靠近List.

# actor
http://www.cnblogs.com/vikings-blog/p/3942417.html

# 函数和方法,def和val区别:
http://www.jianshu.com/p/9b9519a36d78

经测试,scala是从Function0定义到了Function22;
而没有Function23,因此函数对象最多22个参数.


# object和class的区别

1. `class`中均为实例成员函数\变量;
2. `object`中均为static函数变量,为单例模式使用.
若`object`还与类同名,则为伴生对象,类似于友元,还可以访问私有成员.
3. 声明一个object， 一个匿名类就会被创建。

使用方法:
> 1. 成员函数,变量: new一个对象出来调用;
2. 静态函数,变量,或者想使用单例模式: 在同文件中写一个object,然后通过object来定义静态成员,并写使用接口. 如果需要访问私有成员,则这个`object`需要和class同名.

总结:
1. `class`: 实例成员的集合
2. `object`:  静态成员的集合
3. 单例: private构造函数,然后创建`object`. 因为需要访问构造函数,所以其实真正能用的单例对象都是同名的伴生对象. 非同名的object只能称为未关联class的object,可以作为main程序调用. 
4. 访问私有成员: 创建同名`object`(伴生对象)


# scala类修饰符
```
//1.使用var声明field,则该field将同时拥有getter和setter 
class MongoClient(var host:String, var port:Int)

//2.使用val声明field,则该field只有getter。这意味着在初始化之后，你将无法再修改它的值。 
class MongoClient(val host:String, val port:Int)

//3.如果没有任何修饰符，则该field是完全私有的。 
class MongoClient(host:String, port:Int)
```
但是注意对于Case Class，则稍有不同！Case Class对于通过Primary Constructor声明的字段自动添加val修饰，使之变为只读的。


# 主构造函数：Primary Constructor
scala的主构造函数指的是在定义Class时声明的那个函数. 函数的参数就是类的参数,函数体就是类定义中可执行的部分. 
(感觉连类也像是一个函数)
```
class MongoClient(var host: String, var port: Int) {
  def this() = { // 重载构造函数
    this(null, 0)
    host = "127.0.0.1"
    port = 27017

  }
}
```

# Trait 特征
> 带有方法实现\字段的接口,没有参数 (使用的extends关键字,因此更像是抽象类)
> “堆栈化”（叠加）继承特性: 从左到右入栈,从右到左出栈解析;忽略已经出现的方法.(右侧优先深度优先)

- 特征构造顺序(与序列化相反)
```
父类构造器
特征构造器(从右到左)
每个特征当中，父特征先被构造
子类构造器
```
Trait与接口的区别在于,Trait可以直接添加到对象上,而不用在`class`中声明.
- 示例如下
```

trait Debugger {
  def log(message: String) {
    println(message)
  }
}

class Worker {
  def work() {
    println("working innocently")
  }
}

object TestLogger extends App {
  val worker = new Worker() with Debugger // 这里直接附加trait
  worker.work()
  worker.log("log here")
}
```





# Case Class
主构造函数的所有参数会被自动限定为val,也就意味着这个字段是外部可读的。
`Case Class`可以方便得用于DTO(或者叫V0,数据传输对象)的创建,当你希望设计一个类只是作为数据载体的时候.

scala编译器会为case class自动生成一个伴生对象,为class生成4个常用方法:
```
equals
hashcode
toString
copy
// copy方法会基于当前实例的所有字段值复制一个新的实例并返回
```
为伴生对象object生成2个常用方法:
```
apply
unapply
```
上述行为的实现,从编译后的代码看,似乎只写了这些:
```
with scala.Product with scala.Serializable
```
可能这些方法就在Product和Serializable里? 点进去看是空的.



- sealed关键字
```
//sealed关键字声明其他trait都不能再继承当前的trait
//除非这个类与声明的这个trait在同一个class文件里!
sealed trait QueryOption
case object NoOption extends QueryOption
case class Sort(sorting: DBObject, anotherOption: QueryOption) extends QueryOption
case class Skip(number: Int, anotherOption: QueryOption) extends QueryOption 
case class Limit(limit: Int, anotherOption: QueryOption) extends QueryOption
```
case class的companion object是隐式自动生成的
```
case class Person(firstName:String, lastName: String)

//注意：case class的companion object是隐式自动生成的！下面的代码并不是手动编写的。
object Person {
    def apply(firstName:String, lastName:String) = {
        new Person(firstName, lastName)
    }
    def unapply(p:Person): Option[(String, String)] =
        Some((p.firstName, p.lastName))
}
```

# unapply
Scala 提取器是一个带有unapply方法的对象。unapply方法算是apply方法的反向操作：unapply接受一个对象，然后从对象中提取值，提取的值通常是用来构造该对象的值。


# 泛型之 Invariant 与 Covariance/Contravariance
> 默认情况下,scala的泛型是Invariant,也就是强类型,不允许自动上下转型.

```
// +A 表示自动允许向上转型
// -A 表示接受向下转型
case class Item[+A](a: A) { def get: A = a }
val c: Item[Car] = new Item[Volvo](new Volvo) 
// 用父类指针接了子类

// -A 表示接受向下转型
case class Check[-A]() {def check(a: A) {}}
val d:Check[VolvoWagon]= new Check[Volvo]() 
// 用子类指针接了父类
```


# Null/None/Nothing/Nil
>  
1. `null`是`Null`的唯一对象。
`Null`是所有引用类型的子类,
`Any`<-`AnyRef`<-`Null`;
2. `Nothing`是所有类型的子类.返回值是`Unit`表示没有返回值,返回值是`Nothing`表示不但没有返回值,而且还不返回,要想离开这个函数只能抛异常;
```
如果一个函数可以返回Int,也可以抛异常,则这个函数的返回类型写Int即可. 因为Nothing也是Int的子类, 返回类型可以接受向上转型.
```

3. `None`是`Option`的子类. `Option`<-`Some`/`None`;
4. `Nil`是所有List[T]的子类,表示空列表. (List的声明为List[+A],接受向上转型)



# type 关键字 
> scala里的类型，除了在定义class,trait,object时会产生类型，还可以通过type关键字来声明类型。
```
def typeOf[T](v: T): String = v match {
    case _: Int => "Int"
    case _: String => "String"
    case _ => "Unknown"
  }

  def test1(): Unit = {
    type S = String
    val objS:S="string of S"
    println(typeOf(objS))
    println(classOf[S])
  }
```



# <: 与 >: 
类型上确界和类型下确界
```
// T <: Animal的意思是：T必须是Animal的子类(<=)
def biophony[T <: Animal](things: Seq[T]): Unit = things foreach (_.sound())

// >: 传入 Animal的父类或自身, 此时若传入子类,则会被自动向上转型成Animal指针,但实际调用的依然是子类方法
def biophony[T >: Animal](things: Seq[T]) = things

```
# Mutable 和 Immutable 
>Mutable: 被修改时返回原对象引用
Immutable: 被修改时返回新对象的引用
