title: maven3笔记
date: 2015-10-02 17:44:48
tags:
- 配置
categories:
- 配置

---


1. 从maven官网下载maven压缩包，解压缩，配置`MAVEN_HOME`；
2. 最新版的eclipse(mars) j2ee版本直接集成了maven插件m2e；
3. `[ERROR] No plugin found for prefix 'archetype' in the current project and ..`
在`conf/setting`中加入镜像地址：
```
    <mirror> 
           <id>ibiblio.org</id> 
           <name>ibiblio Mirror of http://repo1.maven.org/maven2/</name> 
           <url>http://mirrors.ibiblio.org/pub/mirrors/maven2</url> 
           <mirrorOf>central</mirrorOf> 
           <!-- United States, North Carolina --> 
    </mirror>
    <mirror>  
         <id>jboss-public-repository-group</id>  
         <mirrorOf>central</mirrorOf>  
         <name>JBoss Public Repository Group</name>  
         <url>http://repository.jboss.org/nexus/content/groups/public</url>  
    </mirror>  
```
4. 若maven的conf下`setting.xml`文件中修改了本地仓库reposity的路径，则在eclipse中也要在`windows->prefrence->maven->user settings`中相应修改路径。

参考链接:
http://www.cnblogs.com/leiOOlei/p/3376155.html
http://blog.csdn.net/hay24/article/details/8246203
http://mvnrepository.com/artifact/org.springframework/spring-context
