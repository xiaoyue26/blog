---
title: java代理汇总
date: 2018-07-08 20:24:16
tags: 
- java
- 设计模式
categories:
- java
- 设计模式

---


# 1. 静态代理
1. 接口A;
2. 实现a1,实现A中的方法;
3. 代理a2,
(`AProxy implement A`),成员中有a1,这样实际干活的是a1,但是a2可以夹带一些私货,比如打上日志什么的。
```java
public class AProxy implements AInterface {
    private A1 a1;
 
    public AProxy(A1 a1) {// 包裹一个实现
        this.a1 = a1;
    }
 
    public void doSomething() {
        System.out.println("代理类方法，进行了增强。。。");
        System.out.println("事务开始。。。");
        // 调用委托类的方法;
        a1.doSomething();
        System.out.println("处理结束。。。");
    }
 
}
```

# 2. jdk动态代理
上述静态代理的特点是，如果有n个接口要代理，那么相应的静态代理a2也会有很多。
动态代理的方法就是用反射来动态生成一个代理。而不用每次写非常相似的代码。
1. 接口A;
2. 实现a1,实现A中的方法;
3. 动态代理,传入a1,通过反射创建一个与a1接口一样的类A2,并生成一个对象a2。
然后定义每次invoke的时候需要夹带的私货(打日志)。

外界使用的时候,比静态代理多一步。
静态代理: 1.传入a1,得到一个代理a2.
动态代理: 2.传入a1,得到一个handler,再从handler中获取代理a2.（getProxy）


```java


import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;
import java.lang.reflect.Proxy;

public class DynamicProxyTest {
    private static class AHandler implements InvocationHandler {

        // 目标对象
        private Object a1Impl;

        public AHandler(Object a1Impl) {
            super();
            this.a1Impl = a1Impl;
        }

        /**
         * 创建代理实例
         *
         * @return proxy Object
         * @throws Throwable e
         */
        public Object getProxy() throws Throwable {
            // 实现1: 进行代理:
            return Proxy.newProxyInstance(Thread.currentThread()
                    .getContextClassLoader(), this.a1Impl.getClass()
                    .getInterfaces(), this);
            // 实现2: 不进行代理,依然使用默认实现:
            // 这样写只返回了目标对象，没有生成代理对象。
            // return a1Impl;
        }

        /**
         * 实现InvocationHandler接口方法
         * 执行目标对象的方法，并进行增强
         */
        @Override
        public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
            // 使用代理proxy,调用方法method,参数为args
            Object result;
            System.out.println("代理类方法，进行了增强。。。");
            System.out.println("事务开始。。。");
            // 执行目标方法对象
            // 调用a1Impl的method方法,参数为args
            result = method.invoke(a1Impl, args);
            System.out.println("事务结束。。。");
            return result;
        }
    }

    public static void main(String[] args) throws Throwable {
        A a1 = new A1();
        AHandler handler = new AHandler(a1);
        // 根据目标生成代理对象
        A a2 = (A) handler.getProxy();
        a2.doSomething();
    }

}

```


# 3. cglib动态代理


上述jdk的静态代理和动态代理,本质上都是用接口指针存放实现的对象,然后偷偷用包装了a1的代理a2,替换指针里的对象。
初始化时,代理a2的接口声明和a1一样即可。
> 因此jdk的静态代理和动态代理都要求被代理的实现对象声明实现了某一个接口。

cglib代理则不同,用的是基类指针存放子类对象,因此并不要求一定要声明实现某一个接口,但必须要是一个可以被继承的类。对于java来说，也就是不能是final类。
> jdk代理: 接口引用存放代理对象; 要求实现某接口.
cglib代理: 原类的引用存放代理对象; 要求不是final,可以继承。

具体cglib代理的写法:
```xml
<!-- https://mvnrepository.com/artifact/cglib/cglib -->
<dependency>
    <groupId>cglib</groupId>
    <artifactId>cglib</artifactId>
</dependency>
```

```java
public class AImpl {// 并不继承接口,但不是final的(可以有子类)

    public void doSomething() {
        System.out.println("AImpl doSomething");
    }
}

import java.lang.reflect.Method;
import net.sf.cglib.proxy.Enhancer;
import net.sf.cglib.proxy.MethodInterceptor;
import net.sf.cglib.proxy.MethodProxy;

/**
 * @author xiaoyue26
 */
public class AInterceptor implements MethodInterceptor {
    private Object target;

    /**
     * 创建代理实例
     *
     * @param target
     * @return
     */
    public Object getInstance(Object target) {
        this.target = target;
        Enhancer enhancer = new Enhancer();
        enhancer.setSuperclass(this.target.getClass());// 继承传入的对象
        // 设置回调方法
        enhancer.setCallback(this);
        // 创建代理对象
        return enhancer.create();
    }

    /**
     * 实现MethodInterceptor接口要重写的方法。
     * 回调方法
     */
    @Override
    public Object intercept(Object obj, Method method
            , Object[] args, MethodProxy proxy) throws Throwable {
        System.out.println("事务开始。。。");
        Object result = proxy.invokeSuper(obj, args);// 调用父类对象(调用原实现)
        System.out.println("事务结束。。。");
        return result;
    }

    public static void main(String[] args) {
        AInterceptor aInterceptor = new AInterceptor();
        AImpl a1 = (AImpl) aInterceptor.getInstance(new AImpl());
        a1.doSomething();
    }


}

```

与jdk动态代理很相似,只是概念术语换一下:
创建一个方法拦截器,传入实现a1,获取一个代理a2。
