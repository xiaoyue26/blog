---
title: springboot包结构
date: 2019-09-08 18:32:14
tags: 
- java
- spring
- maven
categories: 
- java
- spring
---

# 使用springboot打包插件
如果用springboot插件进行打包以后,包结构会发生变化:
```xml
<plugin>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-maven-plugin</artifactId>
    <configuration>
        <mainClass>com.tencent.xxx.Application</mainClass>
    </configuration>
    <executions>
        <execution>
            <goals>
                <goal>repackage</goal>
            </goals>
        </execution>
    </executions>
</plugin>
```

springboot打包的jar包用`tar -xvf`解压以后,大致结构如下:
```
 +-META-INF
 |  +-MANIFEST.MF
 |  +-maven
 |     +-pom.xml
 |     +-pom.properties
 +-org
 |  +-springframework
 |     +-boot
 |        +-loader
 |           +-<spring boot loader classes>
 +-BOOT-INF
    +-classes
    |  +-com
    |     +-tencent
    |        +-xxx.class
    |  +-其他src/main/resource路径下的文件
    +-lib
       +-依赖的jar包
```
可以看出主要分为三部分:
`META-INF`文件夹: 元数据信息;
`org`文件夹: springboot框架相关的class和依赖;
`BOOT-INF`: 我们写的代码、resource以及引入的相关依赖。

其中比较重要的是元数据信息中的`MANIFEST.MF`:
```yml
Manifest-Version: 1.0
Archiver-Version: Plexus Archiver
Built-By: mengqifeng
Start-Class: com.tencent.xxx.Application
Spring-Boot-Classes: BOOT-INF/classes/
Spring-Boot-Lib: BOOT-INF/lib/
Spring-Boot-Version: 2.2.0.M4
Created-By: Apache Maven 3.5.3
Build-Jdk: 1.8.0_161
Main-Class: org.springframework.boot.loader.JarLauncher
```
这里可以看出springboot的入口类是`org.springframework.boot.loader.JarLauncher`。
先启动它这个类(main-class)，然后反射调用我们的类(start-class)。

此外由于这里看出lib文件夹的目录是`/BOOT-INF/class/lib`,我们可以手动在pom文件中修改resource文件的打包路径,对准这个目录放进去就可以作为库文件依赖了:
```xml
<resource>
    <directory>src\main\resources\lib</directory>
    <targetPath>/BOOT-INF/class/lib</targetPath>
    <includes>
        <include>**/*.jar</include>
    </includes>
</resource>
```

# 默认的maven包结构
如果把`xxx.jar.original`解压开的话,能得到springboot`repackage`以前的包结构。
此时只有两部分,元数据信息和我们写的代码(字节码和资源文件),没有依赖库:
```
 +-META-INF
 |  +-MANIFEST.MF
 |  +-maven
 |     +-pom.xml
 |     +-pom.properties
 +-com(我们写的代码)以及其他src/main/resource路径下的文件
```

元数据信息中的`MANIFEST.MF`内容也少一些:
```yml
Manifest-Version: 1.0
Archiver-Version: Plexus Archiver
Built-By: mengqifeng
Created-By: Apache Maven 3.5.3
Build-Jdk: 1.8.0_161
```

