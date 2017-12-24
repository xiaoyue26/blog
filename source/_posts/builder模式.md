---
title: builder模式
date: 2016-12-23 19:09:40
tags: 
- java 
- 设计模式
categories:
- java
- 设计模式

---

构造一个对象时，若参数有多个，且其中几个为可选，有几种实现方式:

- 重叠构造函数(重叠构造器)
```java
public NutritionFacts(int a,int b){this(a,b,0);}#调用底层构造函数
public NutritionFacts(int a,int b,int c){
 this.a=a;
 this.b=b;
 this.b=b;
}
```
这种方法较为繁琐，颠倒其中两个参数顺序会造成严重后果。不易扩展。

---

- javeBeans模式
    使用多个setter方法来设定必要的参数，可读性很强，但同时也是javabean容易处于不一致状态、不可用状态，需要额外努力来保证线程安全要求。

---

- builder模式
```java
import java.lang.reflect.Field;

public class NutritionFacts {

	private final int servingSize;
	private final int serving;
	private final int calories;
	private final int fat;
	private final int sodium;
	private final int carbohydrate;

	public static class Builder {
		// Required parameters:
		private final int servingSize;
		private final int serving;
		// optional parameters:
		private int calories = 0;
		private int fat = 0;
		private int sodium = 0;
		private int carbohydrate = 0;

		public Builder(int servingSize, int serving) {
			this.servingSize = servingSize;
			this.serving = serving;
		}

		public Builder calories(int val) {
			calories = val;
			return this;
		}

		public Builder fat(int val) {
			fat = val;
			return this;
		}

		public Builder sodium(int val) {
			sodium = val;
			return this;
		}
		public Builder carbohydrate(int val) {
			carbohydrate = val;
			return this;
		}
		public NutritionFacts build() {
			return new NutritionFacts(this);
		}
	}
	private NutritionFacts(Builder builder) {
		this.servingSize = builder.servingSize;
		this.serving = builder.serving;
		this.calories = builder.serving;
		this.fat = builder.serving;
		this.sodium = builder.serving;
		this.carbohydrate = builder.carbohydrate;

	}
	public static void main(String[] args) {
		NutritionFacts cocacola = new NutritionFacts.Builder(240, 8).calories(100).sodium(35).carbohydrate(27).build();
		 
	}
}

```

builder模式代码比较冗长，但是当大部分参数都是可选的时候是一个不错的选择，拓展性强，比javaBean安全。


- 其他技巧
避免创建不必要的对象，使用静态工厂方法。例如能使用Boolean.valueOf(String)时，就不要使用Boolean(String)。后者会多创建一个对象。

- 内存泄露
消除过期的对象引用：
```java
//自己实现的容器如栈：
private Object[]elements;
public Object pop()//此函数存在内存泄露
{
    if(size==0)
        throw new EmptyStackException();
    return elements[--size];
}
//正确写法:
public Object pop()//此函数存在内存泄露
{
    if(size==0)
        throw new EmptyStackException();
    Object res=elements[--size];
    elements[size]=null;//使废弃对象的引用数减至0.让GC回收。
    return res;
}
```
其他可能出现内存泄露的来源：监听器、回调、缓存。

引用：
1.强引用
强引用就是普通的Java引用，代码：
`StringBuffer buffer = new StringBuffer(); `
2.软引用（SoftReference）//用于缓存
    如果一个对象只具有软引用，则内存空间足够，垃圾回收器就不会回收它；如果内存空间不足了，就会回收这些对象的内存。只要垃圾回收器没有回收它，该对象就可以被程序使用。软引用可用来实现内存敏感的高速缓存（下文给出示例）。
    软引用可以和一个引用队列（ReferenceQueue）联合使用，如果软引用所引用的对象被垃圾回收器回收，Java虚拟机就会把这个软引用加入到与之关联的引用队列中。

3.弱引用（WeakReference）
    弱引用与软引用的区别在于：弱引用的对象拥有更短暂的生命周期。在垃圾回收器线程扫描它所管辖的内存区域的过程中，一旦发现了只具有弱引用的对象，不管当前内存空间足够与否，都会回收它的内存。不过，由于垃圾回收器是一个优先级很低的线程，因此不一定会很快发现那些只具有弱引用的对象。
    >当你想引用一个对象，但是这个对象有自己的生命周期，你不想介入这个对象的生命周期，这时候你就是用弱引用。
    ```java
    String abc = new String("abcde");
    WeakReference<String> wf= new WeakReference<String>(str, rq);
    String abc1 = wf.get()//如果abcde这个对象没有被垃圾回收器回收，那么abc1就指向"abcde"对象
    ```
4.虚引用（PhantomReference）//它允许你知道对象何时从内存中移除。
    “虚引用”顾名思义，就是形同虚设，与其他几种引用都不同，虚引用并不会决定对象的生命周期。如果一个对象仅持有虚引用，那么它就和没有任何引用一样，在任何时候都可能被垃圾回收器回收。
虚引用主要用来跟踪对象被垃圾回收器回收的活动。虚引用与软引用和弱引用的一个区别在于：虚引用必须和引用队列（ReferenceQueue）联合使用。当垃圾回收器准备回收一个对象时，如果发现它还有虚引用，就会在回收对象的内存之前，把这个虚引用加入到与之 关联的引用队列中。

三种类型的引用定义了三种不同层次的可达性级别，由强到弱排列如下：

　　SoftReference > WeakReference > PhantomReference


