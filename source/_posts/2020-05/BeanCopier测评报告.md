---
title: BeanCopier测评报告
date: 2020-05-05 17:38:53
tags:
- spring
- java
categories: 
- java
- spring

---

# What: BeanCopier是什么?
本文讨论的`BeanCopier`具体指的是:
`org.springframework.cglib.beans.BeanCopier`
, 此外用于对比的`BeanUtils`指的是:
`org.springframework.beans.BeanUtils`

其中spring使用`5.1.1.release`,所以cglib版本为`3.2.10`.

BeanCopier和BeanUtils都能用于对象之间浅拷贝成员字段。

# Why: 背景
这里引用一下网上的说法:

> 在做业务的时候，我们有时为了隔离变化，会将DAO查询出来的Entity，和对外提供的DTO隔离开来。大概90%的时候，它们的结构都是类似的，但是我们很不喜欢写很多冗长的b.setF1(a.getF1())这样的代码，于是我们需要BeanCopier来帮助我们。选择Cglib的BeanCopier进行Bean拷贝的理由是，其性能要比Spring的BeanUtils，Apache的BeanUtils和PropertyUtils要好很多，尤其是数据量比较大的情况下。

# 性能测评
参考网上一些benchmark，如https://juejin.im/post/5dc2b293e51d456e65283e61
beanCopier能比beanUtils快30～45倍。

| 场景                                 | 耗时        | 原理       |
|--------------------------------------|-------------|------------|
| 直接使用get&set方法                  | 22ms        | 直接调用   |
| 使用BeanCopiers（不使用Converter）   | 22ms        | 修改字节码 |
| 使用BeanCopiers（使用Converter）     | 249ms       | 修改字节码 |
| 使用BeanUtils                        | 12983ms     | 反射       |
| 使用PropertyUtils（不使用Converter） | 3922ms      | 反射       |

因此如果我们不使用类型转换，使用BeanCopiers几乎没有性能损耗。这是因为cglib修改生成的字节码和get&set几乎是一样的：

```java
public class MA$$BeanCopierByCGLIB$$d9c04262 extends BeanCopier {
    public MA$$BeanCopierByCGLIB$$d9c04262() {
    }
 
    public void copy(Object var1, Object var2, Converter var3) {
        MA var10000 = (MA)var2;
        MA var10001 = (MA)var1;
        var10000.setBooleanP(((MA)var1).isBooleanP());
        var10000.setByteP(var10001.getByteP());
        var10000.setCharP(var10001.getCharP());
        var10000.setDoubleP(var10001.getDoubleP());
        var10000.setFloatP(var10001.getFloatP());
        var10000.setId(var10001.getId());
        var10000.setIntP(var10001.getIntP());
        var10000.setLongP(var10001.getLongP());
        var10000.setName(var10001.getName());
        var10000.setShortP(var10001.getShortP());
        var10000.setStringP(var10001.getStringP());
    }
}
```

## 自测

|                                      | 1kw次       | 1亿次 |
|--------------------------------------|-------------|-------|
| beanUtils                            | 8秒         | 91秒  |
| beanCopier（无converter/有缓存）     | 0.5秒       | 4秒   |
| beanCopier（无converter/无缓存）     | 1.1秒       | 10秒  |
| beanCopier（无converter/懒汉式缓存） | 3.3秒       | 30秒  |


其中各个测试的相关代码:

```java
// 1. beanUtils:
BeanUtils.copyProperties(bean, vo);
 
// 2. beanCopier（无converter/有缓存）:
public static final BeanCopier MODEL_2_VO = BeanCopier.create(Banner.class
            , BannerVO.class, false);
copier.copy(bean, vo, null);
 
// 3. beanCopier（无converter/无缓存）:
copier = BeanCopier.create(Banner.class, BannerVO.class, false);
copier.copy(bean, vo, null);
 
// 4. beanCopier（无converter/懒汉式缓存）:
public static final Map<String, BeanCopier> MAP = new ConcurrentHashMap<>();
copier = MAP.computeIfAbsent(key, k -> BeanCopier.create(Banner.class
                , BannerVO.class, false));
copier.copy(bean, vo, null);

```

## 耗时组成
BeanUtils耗时组成:(主要为反射)

{% img /images/2020-05/beanUtils.png 800 1200 beanUtils %}

BeanCopier（有缓存、无convert）耗时组成:（主要为调用构造函数(xxx::new)）

{% img /images/2020-05/beanCopier1.png 800 1200 beanCopier1 %}

beanCopier（无converter/懒汉式缓存）: 生成key和查询缓存花费了大量的时间，因此第四种写法是得不偿失的。

{% img /images/2020-05/beanCopier2.png 800 1200 beanCopier2 %}

### 总结

BeanCopier（无convert、有缓存）: 主要耗时是业务自身的代码(创建对象)，性能最优，可以考虑;

BeanCopier（无convert、无缓存）：不需要预创建，写法简洁，耗时增加不多，可以考虑。

BeanUtils: 反射调用占用了60%的代码，其中还涉及到查询concurrentHashMap中的bean定义，损耗较大。

# How: 用法
BeanCopier： 只拷贝名称和类型都相同的属性, 基本类型和装箱类型视为不同类型。

如果不符合上述规则，可以自定义converter。（否则可以将converter字段传null）

示例代码:

```java
public static final BeanCopier MODEL_2_VO = BeanCopier.create(Banner.class
            , BannerVO.class, false); // 可以复用一个copier，提高一倍速度
 
banner = ... ; // 例如从DAO获取到
BannerVO vo = new BannerVO();
MODEL_2_VO.copy(b, vo, null); // converter可以直接传null

```


## 支持功能

| 情况                                           | Apache BeanUtils             | Cglib BeanCopier               | Spring BeanUtils |
|------------------------------------------------|------------------------------|--------------------------------|------------------|
| 非public类                                     | 不支持                       | 支持                           | 支持             |
| 基本类型与装箱类型，int->Integer，Integer->int | 支持，可以copy               | 不支持，不copy                 | 不支持，不copy   |
| int->long，long->int，int->Long，Integer->long | 不支持                       | 不支持                         | 不支持           |
| 源对象相同属性无get方法                        | 不支持 不copy                | 不支持 不copy                  | 不支持 不copy    |
| 目标对象相同属性无get方法                      | 支持                         | 不支持                         | 支持             |
| 目标对象相同属性无set方法                      | 不copy，不报错               | 报错                           | 不copy，不报错   |
| 源对象相同属性无set方法                        | 支持                         | 支持                           | 支持             |
| 目标对象相同属性set方法返回非void              | 不设置，其他正常属性可以copy | 不设置，导致其他属性都无法copy | 支持，能够copy   |
| 目标对象多字段                                 | 支持                         | 支持                           | 支持             |
| 目标对象少字段                                 | 支持                         | 支持                           | 支持             |

此外一些较为复杂的情况BeanCopier会进行浅拷贝：

1.属性为对象；

2.属性为List<自定义类>；(注意范型的类型擦除)

当然前提还是源类和目标类中该属性的类型相同，如果不同只能自定义converter了。相应生成的字节码:
```java
public void copy(Object var1, Object var2, Converter var3) {
        BeanB var10000 = (BeanB)var2;
        BeanA var10001 = (BeanA)var1;
        var10000.setAList(((BeanA)var1).getAList());
        var10000.setName(var10001.getName());
    }

```
因此不能用BeanCopier做深拷贝。

对应我们考虑的场景，entity和VO之间拷贝数据，由于entity和VO一般不包含集合或者对象，而且没有修改数据的副作用，因此还是可以用的。


# 线程安全
## copy方法
BeanCopier实例的copy方法是线程安全的，因为它是无状态的，相关讨论：https://cglib-devel.narkive.com/2cqPSUM1/cglib-and-thread-safeness

## create方法
BeanCopier的create方法底层会缓存生成过的字节码，因此不是无状态的，但是有用到synchronized进行线程安全的保护：
```java
protected Object create(Object key) {
    Class gen = null;
synchronized (source) {
        ClassLoader loader = getClassLoader();
        Map cache2 = null;
        cache2 = (Map) source.cache.get(loader);
...
    }
    /** 3.根据生成类，创建实例并返回 **/
    return firstInstance(gen);
}

```

由于BeanCopier的create方法需要查询底层map中的缓存，因此当它生成过的copier非常多的时候，有理由猜测create性能会下降。

1.create方法由悲观锁(synchronized)保护: 并发高时，性能下降；

2.create方法底层有存储: 历史上生成过的copier非常多时，查询性能下降。


# 类卸载

资料2显示，BeanCopier增强的字节码缓存由一个两级map保存，第一级为WeakHashMap，第二级为HashMap，线程安全由synchronized保护。

第一级weakHashMap的key是classloader，因此类的卸载当classloader被回收时进行。

但类似的，如果是我们自己封装拷贝函数，也会面临字节码回收、metaspace占用的问题。



个人认为BeanCopier生成的字节码并不比自己手写的多很多，因此推荐使用BeanCopier。



可能的坑:

跨多个classloader的情况：https://stackoverflow.com/questions/20816197/use-cglib-beancopier-with-multiple-classloaders

BeanCopier无法判断两个不同classloader加载的同名类是不同的类。所以如果使用不同classloader加载同名类，需要特别考虑。



# 参考资料

https://www.cnblogs.com/winner-0715/p/10117282.html

https://www.jianshu.com/p/f8b892e08d26

https://www.cnblogs.com/mengdd/p/3594608.html

https://blog.csdn.net/xihuanyuye/article/details/89887913

https://ningyu1.github.io/blog/20190322/113-object-copy.html