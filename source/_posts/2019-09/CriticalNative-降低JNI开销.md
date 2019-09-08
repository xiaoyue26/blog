---
title: 'CriticalNative:降低JNI开销'
date: 2019-09-01 16:41:03
tags: 
- java
- JNI
categories: 
- java
- 性能
 
---

# 引子
`Android`中有`@CriticalNative`注解:
https://source.android.google.cn/devices/tech/dalvik/improvements
里面说到:
> @FastNative 可以使原生方法的性能提升高达 2 倍，@CriticalNative 则可以提升高达4倍。 

那么这是怎么做到的呢？

# native方法
调用native方法时,JVM的工作步骤:
(源码: http://hg.openjdk.java.net/jdk8u/jdk8u/hotspot/file/4d9931ebf861/src/cpu/x86/vm/sharedRuntime_x86_64.cpp#l1723)
> 1. 创建栈帧;
2. 根据ABI移动参数到寄存器或者栈;(ABI: 应用二进制接口)
3. 封装对象引用到JNI handlers;
4. 获取静态方法的`JNIEnv*`和`jclass`,把他们作为额外参数传递;
5. 检查是否调用`method_entry`; 
6. 检查是否调用对象锁;（`synchronized`）(optinal)
7. 检查native方法是否已经链接;(懒加载函数检查、链接)
8. 线程状态从`in_java`转变为`in_native`;
9. 调用native方法;
10. 检查是否需要safepoint;
11. 线程状态转回`in_java`;
12. 解锁对象锁;(optional)
13. notify `method_exit`;(optional)
14. 将对象结果解出，重置JNI handlers;
15. 处理JNI异常;
16. 移除栈帧。

开销比简单的C调用更大。

此时，如果是足够简单的native方法,可以用`Critical Natives`来降低开销。

# Critical Native方法
`Critical Natives`方法是需要满足下列约束的`native`方法:
> 1. 必须是static且没有synchronized; (省掉上一节的6、12步)
2. 参数类型必须是基本类型或基本类型的数组;(省掉上一节中的对象相关3、14)
3. 具体实现不能调用JNI函数(也就是不使用`JNIEnv* env`和`jclass cls`,既然不使用就不用传给它了),不能分配java对象或者抛出异常;(省掉上一节中的4、15)
4. 不能运行太长时间.(因为它会阻塞gc)

基于这个原理的话, `critical native`方法比普通`native`方法快的原因其实是节省了一些调用开始和结束的开销，因此如果被调用的方法如果是时间占用的大头的话，其实这个优化幅度就很小了。
反之如果是频繁调用的方法，而且每次调用的数据量很小，此时调用开销和执行开销是同量级，那么累计的优化幅度就会很大。
（比如只是长度为16的数组计算的话，计算力提升可以达到2～3倍。）

满足上述约束以后,`Critical Natives`方法还需要进行下列声明:
> 1. 方法名以`JavaCritical_`开头;
2. 没有额外的`JNIEnv*`和`jclass`参数;(因为是static方法,自然也就没有jobject参数了)
3. java数组传递的时候用两个参数: 数组长度、数组引用(基本类型)。
// 这样不再需要调用`GetArrayLength`、`GetByteArrayElements`等函数。
 
`native`方法示例:
```c
JNIEXPORT jint JNICALL
Java_com_package_MyClass_nativeMethod(JNIEnv* env, jclass klass, jbyteArray array) {
    jboolean isCopy;
    jint length = (*env)->GetArrayLength(env, array);
    jbyte* buf = (*env)->GetByteArrayElements(env, array, &isCopy);
    jint result = process(buf, length);
    (*env)->ReleaseByteArrayElements(env, array, buf, JNI_ABORT);
    return result;    
}
```

`Critical Natives`方法示例:
```c
JNIEXPORT jint JNICALL
JavaCritical_com_package_MyClass_nativeMethod(jint length, jbyte* buf) {
    return process(buf, length);
}
```
样例代码: https://gist.github.com/apangin/af70e39b25e578d13484e937c66c7985

`critical`版本的方法是JIT需要的;
普通`native`版本的方法是编译器需要的;

因此实际用的时候，这俩版本的代码都要写上。

(之所以这么繁琐的原因是这个特性和Unsafe一样是jdk内部使用的,没有公开发布给普通程序员,正式发布估计要到jdk10了)

参考: 
http://cr.openjdk.java.net/~jrose/panama/native-call-primitive.html
http://mail.openjdk.java.net/pipermail/panama-dev/2015-December/000225.html
https://stackoverflow.com/questions/36298111/is-it-possible-to-use-sun-misc-unsafe-to-call-c-functions-without-jni/36309652#36309652
