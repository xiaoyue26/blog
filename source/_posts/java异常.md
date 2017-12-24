---
title: java异常
date: 2016-01-24 19:49:17
tags: java
categories:
- java
---

>类继承关系上:
Throwable分为Error和Exception.

>语义上:
异常分为受检异常和非检异常. 
(checked和unchecked)

#非检异常（unckecked exception）：
>Error 和 RuntimeException 以及他们的子类。

编译器不提示和发现这种异常.不要求在程序处理这些异常. 
因为一般是在代码写得有问题才会出现这种异常. 
如除0:`ArithmeticException`;
类型转换: `ClassCastException`;
数组越界: `ArrayIndexOutOfBoundsException`;
空指针: `NullPointerException`. 

#受检异常
运行环境导致的.
如:
`SQLException` , `IOException`,`ClassNotFoundException`.


# 自定义异常:
按照国际惯例，自定义的异常应该总是包含如下的构造函数：
1.一个无参构造函数
2.一个带有String参数的构造函数，并传递给父类的构造函数。
3.一个带有String参数和Throwable参数，并都传递给父类构造函数
4.一个带有Throwable 参数的构造函数，并传递给父类的构造函数。

```
public class IOException extends Exception
{
    static final long serialVersionUID = 7818375828146090155L;
 
    public IOException()
    {
        super();
    }
 
    public IOException(String message)
    {
        super(message);
    }
 
    public IOException(String message, Throwable cause)
    {
        super(message, cause);
    }
 
     
    public IOException(Throwable cause)
    {
        super(cause);
    }
}
```

# 异常与线程
>Java中的异常是线程独立的，线程的问题应该由线程自己来解决，而不要委托到外部，也不会直接影响到其它线程的执行。

Java程序可以是多线程的。每一个线程都是一个独立的执行流，独立的函数调用栈。如果程序只有一个线程，那么没有被任何代码处理的异常 会导致程序终止。如果是多线程的，那么没有被任何代码处理的异常仅仅会导致异常所在的线程结束。

# try,catch,finally
>finally中的return 会覆盖 try 或者catch中的返回值。
实际上,当try或catch中的代码要离开方法或跳出之前,会暂且按下,去finally去完整执行一圈,因此finally中的控制流会优先表达.因此如果在finally里return 100;则方法的返回值就永远是100,而且不会抛出任何异常.(除非finally里抛了.)