title: java里的一些技巧
date: 2015-10-30 10:17:55
tags:
- java
categories:
- java

---


一开始在`quora`上看到`tricks`:
[https://www.quora.com/What-are-some-cool-Java-tricks][1]
我是拒绝的，然而定语是`cool`，所以我就很`cool`地点进去了。

 - 注解:
 `java6+`以来，注解处理器一直是一个未被充分利用的`trick`，它能在编译期就找出错误，而不是等到运行时(这样会增大找出错误的难度)。
  然后答主`Chris Hansen`还推荐使用[Dragger][2]框架进行依赖注入。
 

 - 注释中运行代码:
 后续的答案除了使用JMX检测死锁的线程、如何序列化一个枚举类等，还有一个是讲如何在注释中插入能运行的部分的：
```java
    public static void main(String... args) {   
    // The comment below is no typo.    
    // \u000d System.out.println("This Comment Executed!"); 
}
```

比如这里注释中的`System.out.println`输出语句就是可以运行的。XD简直是无比邪恶的技巧。其他比较好懂的:

 - 类型推断：
  还有一个`tricks`就是类型推断，例如：
```java
public <T> Set<T> hashSet() {
    return (Set<T>)(new HashSet<T>());
}
{
Set<String> set1 = hashSet();
Set<Long> set2 = hashSet();
}
```
这对于数据类型参数化是很有用的。
 
- 变长参数列表：
```java
 void example(String...strings) {};
```

- 在泛型参数上使用联合：其中`Bar`为接口，`Foo`可为类或接口。
```java
public class Baz<T extends Foo & Bar> {}
```
- 使用标号进行流程控制:
```java
outer: 
    for (i = 0; i < arr.length; i++) {
        for (j = 0; j < arr[i].length;  j++) { 
            if (someCondition(i, j)) { 
                break outer; 
             } 
        }
    }
```

- google java编程风格指南:
[http://www.topthink.com/topic/12834.html][3]


  [1]: https://www.quora.com/What-are-some-cool-Java-tricks
  [2]: http://square.github.io/dagger/
  [3]: http://www.topthink.com/topic/12834.html
