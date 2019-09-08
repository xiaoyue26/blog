---
title: JNI如何优雅引用so文件
date: 2019-09-08 18:55:58
tags: 
- java
- JNI
categories: 
- java
- 性能

---

JNI背景知识参见: http://xiaoyue26.github.io/2019/09/07/2019-09/JNI%E6%80%BB%E7%BB%93/

总之假设我们到了临门一脚想要引用so文件到时候,方法有很多种,大致分为两大类:
1. 预先把so文件部署到运行的机器特定目录,代码里使用绝对路径加载;
2. 把so文件打包到resource目录,运行时用相对路径加载。

主要推荐绝对路径的姿势2和相对路径的姿势5.
## 优劣势:

### 绝对路径: 

- 多个java程序可以引用同一个so文件,不用都打包到jar包里,降低jar包大小;  
- 可以灵活切换so文件实现,不用重新打包jar包, 符合c++中动态链接库的思想。

### 相对路径: 

- 一般一个so文件就一个java程序使用,相对路径用起来省心,不用配置多个运行环境. 
- 比较符合jvm平台无关的思想,当然so文件肯定是平台有关的。一般so文件是某个公开库,不是我们自己写的,也不需要修改其实现。

目前我个人使用的是第5种姿势。

## 绝对路径
姿势1: 直接写死:
```java
System.load("/opt/ld_path/libtest.so");
```
姿势2: 结合环境变量,这里第一行代码可以在运行时由命令`java -Djava.library.path=/opt/ld_path`代替:
```java
System.setProperty("java.library.path", "/opt/ld_path");
try {
    Field sysPath = ClassLoader.class.getDeclaredField("sys_paths");
    sysPath.setAccessible(true);
    sysPath.set(null, null);
    // System.out.println(System.mapLibraryName("dynamic"));
    System.loadLibrary("dynamic");
    // 注意mac需要.dylib结尾的依赖文件
    // linux需要.so结尾的依赖文件
} catch (NoSuchFieldException | IllegalAccessException e) {
    System.out.println("error");
    e.printStackTrace();
}
```

## 相对路径: 仅在ide里可用的方法

> 这两种方法都需要首先把`libdynamic.so`文件放到`resource`目录。

姿势3:
```java
URL url = JNIDyn.class.getClassLoader().getResource("libdynamic.so");
System.load(url.getPath());
```
姿势4:
```java
ClassPathResource resource = new ClassPathResource("libdynamic.so");
// System.out.println(resource.getPath());
try {
    File file = resource.getFile();
    System.load(file.getAbsolutePath());
} catch (IOException e) {
    e.printStackTrace();
}
```

## 相对路径: ide和jar包都能用的方法
姿势5: 

> 需要首先把`libdynamic.so`文件放到`resource`目录。

然后需要创建`NativeUtils`工具类;
加载代码:
```java
static {
    try {
        NativeUtils.loadLibraryFromJar("/libnative.so");
    } catch (IOException e) {
        e.printStackTrace();
    }
}
```
这里使用到的`NativeUtils`源码如下:
```java
import java.io.*;
import java.nio.file.*;
public class NativeUtils {
    private static final int MIN_PREFIX_LENGTH = 3;
    public static final String NATIVE_FOLDER_PATH_PREFIX = "nativeutils";
    private static File temporaryDir;
    private NativeUtils() {}
    public static void loadLibraryFromJar(String path) throws IOException {
        if (null == path || !path.startsWith("/")) {
            throw new IllegalArgumentException("The path has to be absolute (start with '/').");
        }
        String[] parts = path.split("/");
        String filename = (parts.length > 1) ? parts[parts.length - 1] : null;
        if (filename == null || filename.length() < MIN_PREFIX_LENGTH) {
            throw new IllegalArgumentException("The filename has to be at least 3 characters long.");
        }
        if (temporaryDir == null) {
            temporaryDir = createTempDirectory(NATIVE_FOLDER_PATH_PREFIX);
            temporaryDir.deleteOnExit();
        }
        File temp = new File(temporaryDir, filename);
        try (InputStream is = NativeUtils.class.getResourceAsStream(path)) {
            Files.copy(is, temp.toPath(), StandardCopyOption.REPLACE_EXISTING);
        } catch (IOException e) {
            temp.delete();
            throw e;
        } catch (NullPointerException e) {
            temp.delete();
            throw new FileNotFoundException("File " + path + " was not found inside JAR.");
        }
        try {
            System.load(temp.getAbsolutePath());
        } finally {
            if (isPosixCompliant()) {
                temp.delete();
            } else {
                temp.deleteOnExit();
            }
        }
    }

    private static boolean isPosixCompliant() {
        try {
            return FileSystems.getDefault()
                    .supportedFileAttributeViews()
                    .contains("posix");
        } catch (FileSystemNotFoundException
                | ProviderNotFoundException
                | SecurityException e) {
            return false;
        }
    }

    private static File createTempDirectory(String prefix) throws IOException {
        String tempDir = System.getProperty("java.io.tmpdir");
        File generatedDir = new File(tempDir, prefix + System.nanoTime());

        if (!generatedDir.mkdir())
            throw new IOException("Failed to create temp directory " + generatedDir.getName());

        return generatedDir;
    }

}
```

