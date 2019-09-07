---
title: JNI总结
date: 2019-09-07 18:20:10
tags: 
- java
- JNI
categories: 
- java
- 性能

---

# what: JNI是啥
`JNI(Java Native Interface)`是java访问`native`方法的接口规范。
所谓`native`方法一般是c/c++代码。（也可以是汇编）
java实现了一个JNI框架来让java和其他语言互调，java方法可以调JNI接口声明了的native方法，native方法也可以创建、使用java对象。
JNI接口规范主要按照`c`语言，不像`c++`一样改写方法名。
因此实际编码中需要用`extern c`来维持方法名的纯净。


编译方法:
c++: `print(int)`=>`print_int`;
c: `print`. 
所以我们需要c这种风格的。(不支持重载)


# why: 为啥要使用JNI
使用的场景包括:
1. 有些现成的代码是c/c++的，需要在java中调用; （比如一些平台相关的、SIMD操作、或其他java中没有的库）
2. c/c++版本的代码也许有巨大的性能优势。

# HOW: JNI如何工作
## 如何使用JNI
两种方法： 静态注册和动态加载。
### 静态注册
假设我们要在java中调用c的方法,大致分为6个步骤:
1. 在java中声明一个`native`方法,但是不实现;
2. 编译java字节码,生成`class`文件;(`javac`命令)
3. 用class文件生成`.h`的文件头;(`javah`命令)
4. 创建`.c`文件实现`.h`文件头中声明的方法;
5. 编译`.c`，`.h`文件生成动态链接库`.so`;
6. 在`java`中加载`.so`文件,使用第一步中声明的方法。

相关命令:
```shell
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:.
javac HelloWorld.java
javah HelloWorld
gcc -fPIC -I /usr/lib/jvm/jdk/include -I /usr/lib/jvm/jdk/include/linux -shared libHelloWorld.c -o libHelloWorld.so
export LD_LIBRARY_PATH=.:$LD_LIBRARY_PATH # 为了找到so文件
java -Djava.library.path=. HelloWorld  # 也是为了找到so文件(二选一即可)
```

### 动态加载
利用`RegisterNatives`方法来注册Java方法与JNI函数的映射。

1. 利用结构体`JNINativeMethod`数组记录 Java 方法与 JNI 函数的对应关系
2. 实现 `JNI_OnLoad` 方法，在加载动态库后，执行动态注册
3. 调用 `FindClass` 方法，获取Java对象
4. 调用 `RegisterNatives`方法，传入 Java 对象、`JNINativeMethod`;
5. 数组及注册方法数完成注册；

```c++
// 以下是c++版本的,c语言版本的话很简单,只要把:
// env->改成(*env)->
// 调用的方法参数第一个参数加上env即可。
#define JNI_CLASS_PAPT "com/xxx"
JNIEXPORT jstring JNICALL native_test(JNIEnv *env, jobject instance) {
    return env->NewStringUTF("hello world");
}
static JNINativeMethod g_methods[] = {
        // Java层方法、参数类型、native方法
        {"get_hello_world", "()Ljava/lang/String;", (void*)native_test}
};

// 动态库加载时回调方法
jint JNI_OnLoad(JavaVM *vm, void *reserved) {
    JNIEnv *env = NULL;
    vm->GetEnv((void**)&env, JNI_VERSION_1_8);
    jclass clazz= env->FindClass(JNI_CLASS_PAPT);
    
    // 注册Java和natvie方法映射表
    env->RegisterNatives(clazz
    , g_methods
    , sizeof(g_methods)/sizeof(g_methods[0]));
    return JNI_VERSION_1_8;
}

```

参见`jni.h`中的`JNINativeMethod`结构体:
```c
typedef struct {
    const char* name; // java方法名
    const char* signature;// java方法签名
    void*       fnPtr; // c函数指针
} JNINativeMethod;
```


## JNI原理
本质其实就是JVM使用了so动态链接库中的函数，所以关键在于函数名的映射。
一个典型的`native`方法的签名如下:
```c
// native方法的签名由类名(含包名,点换成下划线)和方法名拼接而成:
JNIEXPORT void JNICALL Java_packname_classname_methodname
  (JNIEnv *env, jobject obj)
{
    /*Implement Native Method Here*/
}
```
可见，JVM调用native方法的时候，需要传递一个`JNIEnv`指针和一个`jobject`指针。
> JNIEnv: 包含访问JVM的接口，可以进行native数组和java数组转换，字符串转换，对象实例化、抛异常等等java能做的事情；

> jobject: 声明native方法的java对象。

每一个Java线程对应一个`JNIEnv`。
`JNIEnv`指针仅在native方法当前线程中有效；如果手动保存到其他地方，然后在其他线程中想要使用，需要调用`AttachCurrentThread`来挂靠当前线程到jvm，使用完毕后调用`DetachCurrentThread`脱离jvm。
挂靠样例:
```c
// 1. 
JNIEnv *env;
(*g_vm)->AttachCurrentThread (g_vm, (void **) &env, NULL);
// 2. 脱离:
(*g_vm)->DetachCurrentThread (g_vm);
```



## 类型转换
native和java的基本类型能自动互转，复杂类型（数组、数组、对象）则要使用`JNIEnv`显式地进行转换。

字符串转换(C++版本):
```c++
extern "C"
JNIEXPORT void JNICALL Java_ClassName_MethodName
  (JNIEnv *env, jobject obj, jstring javaString)
{
    // java字符串=>c字符串
    const char *nativeString = env->GetStringUTFChars(javaString, 0);
    // do something with nativeString
    // 释放:
    env->ReleaseStringUTFChars(javaString, nativeString);
}
```
c语言版本就是参数多了`env`参数:
```c
JNIEXPORT void JNICALL Java_ClassName_MethodName
  (JNIEnv *env, jobject obj, jstring javaString)
{
// 转换:
    const char *nativeString = (*env)->GetStringUTFChars(env, javaString, 0);
// 释放:
    (*env)->ReleaseStringUTFChars(env, javaString, nativeString);
}
```

## 基本类型的映射

 | native类型     | Java类型 | 描述       | java类型签名（signature） |  |
|--------|------|--------|---|---|
| unsigned char  | jboolean    | unsigned 8位 | Z   |  |
| signed char   | jbyte     | signed 8位  | B   |  |
| unsigned short  | jchar     | unsigned 16位 | C   |  |
| short      | jshort     | signed 16位  | S   |  |
| long       | jint      | signed 32位  | I   |  |
| long long__int64 | jlong     | signed 64位  | J   |  |
| float      | jfloat     | 32位   | F   |  |
| double      | jdouble    | 64位     | D   |  |
| void       |void| | V   |  |


string类的类型签名: `Ljava/lang/String;`
整型数组的类型签名: `[I`
`int[][]`的签名: `[[I`

# JNI代码中调用java对象方法
## 1. 调用实例方法
首先我们有env和obj，步骤是：
> 1. 用env、obj获取class对象cls;
2. 用env、cls和方法签名反射获得方法引用mid;
3. 用env、obj、mid调用方法。

```c
JNIEXPORT void JNICALL  Java_InstanceMethodCall_nativeMethod(JNIEnv *env, jobject obj) { 
     jclass cls = (*env)->GetObjectClass(env, obj);  
     jmethodID mid =  (*env)->GetMethodID(env, cls, "callback", "()V");  
     if (mid == NULL) { 
         return; /* method not found */ 
     } 
     printf("In C\n"); 
     (*env)->CallVoidMethod(env, obj, mid);  
}
```
## 2. 调用静态方法
前两步和刚才一样,第三部把Obj换成cls即可:
1. 获取cls;
2. 获取mid;
3. 用env、cls、mid调用静态方法。

```c
JNIEXPORT void JNICALL  Java_StaticMethodCall_nativeMethod(JNIEnv *env, jobject obj) { 
     jclass cls = (*env)->GetObjectClass(env, obj); 
     jmethodID mid =  
         (*env)->GetStaticMethodID(env, cls, "callback", "()V"); 
     if (mid == NULL) { 
         return;  /* method not found */ 
     } 
     printf("In C\n"); 
     (*env)->CallStaticVoidMethod(env, cls, mid);  // 这里是cls
} 
```


# JNI需要注意的点
1. native方法自己管理内存,jvm不gc这部分;
2. JNI调用开销较大，不宜频繁调用;（java数组、字符串都会线性拷贝）
3. JNI方法平台有关,移植性差;
4. c代码里显式释放内存;
5. 字符编码问题。

第四点一般是获取和释放成对使用：(多少get就有多少delete或release)
```
GetObjectField=>DeleteLocalRef
GetStringUTFChars=>ReleaseStringUTFChars
```

最后一个字符编码问题:
JNI里的这几个函数实际上用的是修改版本的`UTF-8`，并不完全等效于`UTF-8`：
```
NewStringUTF
GetStringUTFLength
GetStringUTFChars
ReleaseStringUTFChars
GetStringUTFRegion
```
用户应当使用这几个函数,先创建UTF-16，然后安全地转换成标准`UTF-8`。
```
NewString
GetStringLength
GetStringChars
ReleaseStringChars
GetStringRegion
GetStringCritical
ReleaseStringCritical
```

