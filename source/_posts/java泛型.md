---
title: java泛型
date: 2017-02-12 19:41:36
tags: java
categories: java
---

> 上界`<? extends T>`
不能往里存，只能往外取
```
Plate<? extends Fruit>p= new Plate<Apple>(new Apple());
//  赋值后,p为一个容器,容器元素类型为CAP#1,是一种Fruit的子类.

// 写:
//不能存入任何元素:
p.set(new Fruit());//Error 无法确定CAP#1能否接住Fruit.
p.set(new Apple());//Error 

// 读:
// 读出来的东西只能存放在Fruit及其基类中:
Fruit new1=p.get(); // 可以,CAP#1可以存放到Fruit中
Object new2=p.get(); // 可以
Apple new3=p.get();//Error
```

>下界`<? super T>`
不影响往里存，但往外取只能放在Object对象里
```
Plate<? super Fruit>p=new Plate<Fruit>(new Fruit());
//  赋值后,p为一个容器,容器元素类型为CAP#1,是一种Fruit的基类.
// 写:
p.set(new Fruit());// 可以, CAP#1可以接住Fruit
p.set(new Apple());//基类指针可以存放子类对象

// 读:
Apple new1=p.get();//Error 不确定Apple能否接住CAP#1
Fruit new2=p.get();//Error
Object new3=p.get();
```

>PECS原则
（Producer Extends Consumer Super）原则
1. 频繁往外读取内容的，适合用上界Extends。
2. 经常往里插入的，适合用下界Super。