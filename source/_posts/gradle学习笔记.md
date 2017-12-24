---
title: gradle学习笔记
date: 2016-12-31 19:18:04
tags: gradle
categories: gradle
---

有用的命令:
```
// 查看某个依赖具体是怎么引入的.
gradle dI --dependency org.springframework:spring-messaging
// dI是dependencyInsight的缩写
```
https://www.jetbrains.com/help/idea/creating-and-running-your-scala-application.html
https://docs.gradle.org/3.3/userguide/scala_plugin.html
https://dzone.com/articles/intellij-scala-and-gradle
http://www.cnblogs.com/davenkin/p/gradle-learning-4.html

http://www.cnblogs.com/davenkin/p/gradle-learning-1.html
git clone https://github.com/davenkin/gradle-learning.git

# gradle task
gradle主要有两个元素:task和plugin.
如下task开头的就是task,没有task的就是project的属性(或扩展属性).
```
task helloWorld << {
    println "Hello World!"
}

task copyFile(type: Copy) {
    from 'xml'
    into 'destination'
}

task hello2 { //doLast等效于helloWorld的 << 
   doLast {
        println 'hello2'
    }
}
task hello3 {
    doLast {
        println 'hello last'
    }
   doFirst {
      println 'hello first'
    }
}
```
可以看出gradle也是有语法的.据说是groovy:
http://blog.csdn.net/wangbaochu/article/details/51177672
http://www.jianshu.com/p/37b46cc815b3

1. 大括号之间的内容则表示传递给task()方法的一个闭包.
2. << 表示向helloWorld中追加入执行代码——其实就是groovy代码.
3. 执行命令: `gradle helloWorld`,表示执行`helloWorld`这个task.
4. helloWorld是默认类型,copyFile是Copy类型的task.

# task之间的依赖:
```
task taskA(dependsOn: taskB) {
   //do something
}
```

# 查看当前项目的task列表:(包括自带的tasks)
```
gradle tasks --all
```

# 查看当前的属性配置列表:
```
gradle properties
```

#执行顺序:
Gradle在执行Task时分为两个阶段，首先是配置阶段，然后才是实际执行阶段.
```
task hello9 << {
   println description
}

hello9.configure {
   description = "this is hello9"
}
```
配置在定义后,但依然能打印出`this is hello9`.
一个Task除了执行操作之外，还可以包含多个Property，其中有Gradle为每个Task默认定义的Property，比如description，logger等。另外，每一个特定的Task类型还可以含有特定的Property，比如Copy的from和to等。当然，我们还可以动态地向Task中加入额外的Property。

# 增量编译:
```
//规定输入输出:
inputs.dir sources
outputs.file destination
```
# 静态设置变量:
```
version = 'this is the project version'
description = 'this is the project description'
task showProjectProperties << {
    println version // task没有version属性,因此默认调用 project.version
    println project.version
    println description // task有description属性,但是没有赋值
    println project.description

}

// 为project添加额外的Property(不能像python那样直接定义,需要用ext关键字添加)
ext.property1 = "this is property1"

ext {
    property2 = "this is property2"
}

// project和task都是实现了ExtensionAware接口的gradle对象,因此可以这样添加属性
task showProperties << {
    println property1
    println property2
}
```

# 动态设置变量
```
task showCommandLieProperties << {
    println property3
}
```
调用命令:
```
// 动态定义属性(命令行中定义),调用命令:
gradle -Pproperty3="this is property3" showCommandLieProperties
// 或者:
gradle -Dorg.gradle.project.property3="this is another property3" showCommandLieProperties
// 或者:
export ORG_GRADLE_PROJECT_property3="this is yet another property3"
gradle showCommandLieProperties
unset ORG_GRADLE_PROJECT_property3
```

# gradle复合项目
```
//根项目:
//settings.gradle
include 'sub-project1', 'sub-project2'

//作用于所有项目的task:(包括root-project)
allprojects {
    apply plugin: 'idea'
    task allTask << {
        println project.name
    }
}
//作用于所有子项目的task:(不包括root-project)
subprojects {
    task subTask << {
        println project.name
    }
}
//gradle allTask 输出:
:allTask  // 调用的root的allTask
root-project //输出
:sub-project1:allTask //调用sub-project1的allTask
sub-project1
:sub-project2:allTask
sub-project2
```

可以在跟项目的build.gradle中定义所有子项目的task:
```
// 方法1:
project(':sub-project1') {
    task forProject1 << {
        println 'for project 1'
    }
}

// 方法2:
configure(allprojects.findAll { it.name.startsWith('sub') }) {
    subTask << {
        println 'this is a sub project'
    }
}
```
执行`gradle forProject1`时会级联查找所有项目的对应task.也可以显式得调用某个子项目的task:
```
gradle :sub-project1:forProject1
```

groovy语法:
http://blog.csdn.net/u014761700/article/details/51867939

# gradle执行阶段:
Initialization phase
->hook1
->Configuration phase
->hook2
->Execution phase
->hook3

其中hook1:
```
gradle.beforeProject{
project->
    ...
}
```
hook2:
```
gradle.taskGraph.whenReady{
graph->
    ...
}
```
hook3:
```
gradle.buildFinished{
result->
    ...
}
```
1. 初始化阶段执行`settings.gradle`
2. 配置阶段解析每个project中的build.gradle
,以确定整个build的project以及内部的task关系.(即hook2中的task的有向图)
3. 执行阶段

gradle包含三大对象: gradle对象,project对象,task对象.
一个project对象有多少task对象,往往由plugin对象决定. 

spring的gradle:
https://github.com/spring-projects/spring-scala/blob/master/build.gradle

# gradle排除依赖:
```
compile("org.springframework.boot:spring-boot-starter-web") {
        exclude group: 'com.fasterxml.jackson.core'
      } // 排除依赖示例
```
maven也有类似的语法排除.
## 多版本库的选择:
- maven: 后面引入的覆盖前面的.可以使用shadow插件进行换名;
- gradle: 新版本覆盖旧版本.