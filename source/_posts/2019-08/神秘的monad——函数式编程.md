---
title: 神秘的monad——函数式编程
date: 2019-08-23 09:20:21
tags: 
- scala
- 函数式
- monad
categories: 
- java
- 设计模式

---

monad确实比较难理解，我认真翻了一个星期资料才理解。

讲得比较好的参考资料：
http://josephguan.github.io/2016/06/25/monad-in-scala/
比较形象的、有图的：
http://blog.forec.cn/2017/03/02/translation-adit-faamip/
数学上讲得比较多的：(scala版代码可用)
https://segmentfault.com/a/1190000008000905

参考资料3中的scala版代码:
```scala
// 1. 半群:
trait SemiGroup[A] {
  def op(a1: A, a2: A): A
}

// 2. monoid: (还不是monad) 比半群多一个零元
trait Monoid[A] extends SemiGroup[A] {
  def zero: A
}

// 函子: (有map函数就是Functor)
trait Functor[F[_]] {
  // 输入一个A的容器F[A],输入一个A类型=>B类型的容器
  // 输出B类型的容器F[B]
  def map[A, B](a: F[A])(f: A => B): F[B]
}


object MonadTest {
  def main(args: Array[String]): Unit = {
    val stringMonoid = new Monoid[String] {
      def op(a1: String, a2: String) = a1 + a2

      def zero = ""
    }

    def listMonoid[A] = new Monoid[List[A]] {
      def op(a1: List[A], a2: List[A]) = a1 ++ a2

      def zero = Nil
    }

    def optionMonoid[A] = new Monoid[Option[A]] {
      def op(a1: Option[A], a2: Option[A]) = a1 orElse a2

      def zero = None
    }

    def listFunctor = new Functor[List] {
      def map[A, B](a: List[A])(f: (A) => B) = a.map(f)
    }

  }
}


/*trait Monad[M[_]] {
  def unit[A](a: A): M[A]   //identity
  def join[A](mma: M[M[A]]): M[A]
}*/
// Monad: (有unit和flatmap就是monad)
trait Monad[M[_]] {
  def unit[A](a: A): M[A]

  def flatMap[A, B](fa: M[A])(f: A => M[B]): M[B]

  // def join[A](mma: M[M[A]]): M[A] = flatMap(mma)(ma => ma)
}



```


附抄scala版的monad(参考资料1):
```scala
trait Monad[+T] {
  def flatMap[U]( f : (T) => Monad[U] ) : Monad[U]
  def unit(value : B) : Monad[B]
}

// map可以理解为flatmap的特化:
def map[U](f : (T) => U) : Monad[U] = flatMap(v => unit(f(v)))

// Try类型monad:
val result:Try[Int] = Try("5".toInt).flatMap{a =>
                      Try("6a".toInt).flatMap{b =>
                      Try("9".toInt).flatMap{c => Try(a + b +c )}}}
// for是flatmap的语法糖:
val result: Try[Int] = for (
    a <- Try("5".toInt);
    b <- Try("6a".toInt);
    c <- Try("9".toInt)
  ) yield( a + b + c)

// 1.1含幺半群G: 
trait Monad[+T]
// 1.2二元封闭、结合运算:
def flatMap[U]( f : (T) => Monad[U] ) : Monad[U]
// 1.3幺元:
def unit(value : B) : Monad[B] 
// 2.1结合律/封闭:
monad.flatMap(f).flatMap(g) == monad.flatMap(v => f(v).flatMap(g)) // associativity
// 案例:
val multiplier : Int => Option[Int] = v => Some(v * v)
val divider : Int => Option[Int] = v => Some(v/2)
val original = Some(10)

original.flatMap(multiplier).flatMap(divider) ===
original.flatMap(v => multiplier(v).flatMap(divider))
    
// 2.2 左幺元
unit(x).flatMap(f) == f(x)
// 案例:
val multiplier : Int => Option[Int] = v => Some(v * v)
val item = Some(10).flatMap(multiplier)
item === multiplier(10)

// 2.3 右幺元
monad.flatMap(unit) == monad
// 案例:
val value = Some(50).flatMap(v => Some(v))
value === Some(50)

// 3. 范畴
高阶类型（如List[T+]）

// 4. 函子(Functor)
// 函数， Int => String
def foo(i:Int): String = i.toString
// 函子， List[T] => Set[T]
scala> def baz[T](l:List[T]): Set[T] = l.toSet

// 5. 自函子(Endofunctor)：
把一个类型映射到自身类型，比如Int=>Int, String=>String 
例如flatmap:
def flatMap[U]( f : (T) => Monad[U] ) : Monad[U]
```



> 下面开始是我个人的理解

## 函数式语言
> 函数是一等公民。
无函数副作用。

（学校里教的）
函数可以像普通变量一样使用。（比c里的函数指针更进一步）

### 更函数式一点
尽量无状态，最好都像lambda演算一样，有很深的递归。
用递归代替循环。

Monad就是这个思想的一个具体实现。
## 代码层面理解
Monad在scala中就是一个有flatmap的容器，可以把函数fmap的输出收集起来打平回原来的Monad类型。
比较好理解的Monad类型是容器类型：List,Option. 
## 形象上理解
Monad形象上理解类似于有管道操作的容器，可以把函数fmap的输出适配回Monad类型，方便投入下一个函数中。

## 比较严密的定义上理解：
（去掉范畴学的数学术语，简化理解）
Monad是一个我们定义的集合，它上面有零元（如Option中的None\List中的nil），它上面还有一种二元操作op，op(A,B)的结果依然属于Monad(封闭性)，并且运算满足结合律(可以随意加括号)。
所以如果有unit函数(生成零元)，flatmap函数（把二元操作打平回集合元素类型，满足封闭性），就可以成为一个Monad了。至于结合律，由于函数都满足结合律，因此可以忽略。

总结: 
有map函数: Functor、函子
有ap函数（参数为函数的map）: Applicative、应用
有flatmap函数: Monad


