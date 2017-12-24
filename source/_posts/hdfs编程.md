title: hdfs编程
date: 2015-01-03 11:24:46
tags: hdfs
categories:
- java
---


# Eclipse/mvn下使用hdfs的java API

1. 首先下载J2EE版本的Eclipse，这样eclipse里能自带maven，省得自己下载和配置了。
2. 建立mvn工程，或者建立java工程后转为mvn工程。
3. 下载hadoop并解压，记录一下本地库的路径。（我的为`/usr/local/hadoop/lib/native`）由于要使用hdfs的新feature(`hdfs storagepolicies`),所以安装的是目前最新的2.7.1版。
```shell
hdfs storagepolicies -listPolicies
# 结果
Block Storage Policies:
	BlockStoragePolicy{COLD:2, storageTypes=[ARCHIVE], creationFallbacks=[], replicationFallbacks=[]}
	BlockStoragePolicy{WARM:5, storageTypes=[DISK, ARCHIVE], creationFallbacks=[DISK, ARCHIVE], replicationFallbacks=[DISK, ARCHIVE]}
	BlockStoragePolicy{HOT:7, storageTypes=[DISK], creationFallbacks=[], replicationFallbacks=[ARCHIVE]}
	BlockStoragePolicy{ONE_SSD:10, storageTypes=[SSD, DISK], creationFallbacks=[SSD, DISK], replicationFallbacks=[SSD, DISK]}
	BlockStoragePolicy{ALL_SSD:12, storageTypes=[SSD], creationFallbacks=[DISK], replicationFallbacks=[DISK]}
	BlockStoragePolicy{LAZY_PERSIST:15, storageTypes=[RAM_DISK, DISK], creationFallbacks=[DISK], replicationFallbacks=[DISK]}
```
以温数据的策略为例，当前的策略是：把温数据存储到磁盘或归档设备中(`storageTypes=[DISK, ARCHIVE]`)，如果创建文件时无法访问指定的设备，则回退到磁盘或归档设备中(`creationFallbacks=[DISK, ARCHIVE]`)，如果创建副本时无法访问指定设备则回退到磁盘或归档设备中(`replicationFallbacks=[DISK, ARCHIVE]`)。
4.编写`pom.xml`文件：
```xml
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>
  <groupId>Runner</groupId>
  <artifactId>Runner</artifactId>
  <version>0.0.1-SNAPSHOT</version>
  <properties>
		<project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
		<hadoop.version>2.7.1</hadoop.version>
		<jackson.version>2.5.0</jackson.version>
	</properties>
  <dependencies>  
    <dependency>  
      <groupId>junit</groupId>  
      <artifactId>junit</artifactId>  
      <version>3.8.1</version>  
      <scope>test</scope>  
    </dependency>  
    
     <dependency>  
          <groupId>org.apache.hadoop</groupId>  
          <artifactId>hadoop-minicluster</artifactId>  
          <version>${hadoop.version}</version>  
    </dependency>  
    <dependency>  
          <groupId>org.apache.hadoop</groupId>  
          <artifactId>hadoop-client</artifactId>  
          <version>${hadoop.version}</version>  
    </dependency>  
    <dependency>  
          <groupId>org.apache.hadoop</groupId>  
          <artifactId>hadoop-assemblies</artifactId>  
          <version>${hadoop.version}</version>  
    </dependency>  
        <dependency>  
          <groupId>org.apache.hadoop</groupId>  
          <artifactId>hadoop-maven-plugins</artifactId>  
          <version>${hadoop.version}</version>  
    </dependency>  
        <dependency>  
          <groupId>org.apache.hadoop</groupId>  
          <artifactId>hadoop-common</artifactId>  
          <version>${hadoop.version}</version>  
    </dependency>  
        <dependency>  
          <groupId>org.apache.hadoop</groupId>  
          <artifactId>hadoop-hdfs</artifactId>  
          <version>${hadoop.version}</version>  
    </dependency>  
  </dependencies>  
  <build>
    <sourceDirectory>src</sourceDirectory>
    <plugins>
      <plugin>
        <artifactId>maven-compiler-plugin</artifactId>
        <version>3.3</version>
        <configuration>
          <source>1.8</source>
          <target>1.8</target>
        </configuration>
      </plugin>
    </plugins>
  </build>
</project>
```
我是用java工程转的mvn工程，所以可能其他配置有所不同。但`dependency`应该是差不多的。`hadoop`版本为`2.5.2`，可以灵活修改`properties`部分的`hadoop.version`。
5.把之前记录的`hadoop`库地址配置进运行配置里，(Run configuration里在vm arguments里填上：
```shell
-Djava.library.path=/usr/local/hadoop/lib/native
```
)这样虚拟机运行的时候就能去本地库目录下调用`hadoop`的`*.so`等文件了
6.编写`java`文件：
```java
import java.io.BufferedReader;
import java.io.BufferedWriter;
import java.io.File;
import java.io.FileInputStream;
import java.io.FileNotFoundException;
import java.io.IOException;
import java.io.InputStream;
import java.io.InputStreamReader;
import java.io.OutputStreamWriter;
import java.io.Writer;
import java.util.Date;

import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.fs.BlockLocation;
import org.apache.hadoop.fs.FSDataInputStream;
import org.apache.hadoop.fs.FSDataOutputStream;
import org.apache.hadoop.fs.FileStatus;
import org.apache.hadoop.fs.FileSystem;
import org.apache.hadoop.fs.Path;
import org.apache.hadoop.hdfs.DistributedFileSystem;
import org.apache.hadoop.hdfs.DFSClient.*;
import org.apache.hadoop.hdfs.protocol.DatanodeInfo;
import org.apache.hadoop.hdfs.server.namenode.dfsclusterhealth_jsp;

public class HadoopFSOperations {

	private static Configuration conf = new Configuration();
	private static final String HADOOP_URL = "hdfs://feng:9000";
	private static final String ENCODE = "utf-8";
	private static FileSystem fs;

	private static DistributedFileSystem hdfs;

	static {
		try {
			FileSystem.setDefaultUri(conf, HADOOP_URL);
			fs = FileSystem.get(conf);
			hdfs = (DistributedFileSystem) fs;
		} catch (Exception e) {
			e.printStackTrace();
		}
	}

	/**
	 * 列出所有DataNode的名字信息
	 */
	public void listDataNodeInfo() {
		try {
			DatanodeInfo[] dataNodeStats = hdfs.getDataNodeStats();
			String[] names = new String[dataNodeStats.length];
			System.out.println("List of all the datanode in the HDFS cluster:");

			for (int i = 0; i < names.length; i++) {
				names[i] = dataNodeStats[i].getHostName();
				System.out.println(names[i]);
			}
			System.out.println(hdfs.getUri().toString());
		} catch (Exception e) {
			e.printStackTrace();
		}
	}

	/**
	 * 查看文件是否存在
	 */
	public void checkFileExist(String filepath) {
		try {
			Path f = new Path(filepath);
			boolean exist = fs.exists(f);
			System.out.println("Whether exist of this file:" + exist);
		} catch (Exception e) {
			e.printStackTrace();
		}
	}

	public void deleteFile(String filepath) {
		try {
			Path f = new Path(filepath);
			boolean exist = fs.exists(f);
			System.out.println("Whether exist of this file:" + exist);
			// 删除文件
			if (exist) {
				boolean isDeleted = hdfs.delete(f, false);
				if (isDeleted) {
					System.out.println("Delete success");
				}
			} else {
				System.out.println("file not found:" + filepath);
			}
		} catch (Exception e) {
			e.printStackTrace();
		}
	}

	/**
	 * 创建文件到HDFS系统上
	 */
	public void createFile(String filepath, String content) {
		try {
			Path f = new Path(filepath);
			System.out.println("Create and Write :" + f.getName() + " to hdfs");

			FSDataOutputStream os = fs.create(f, true);
			Writer out = new OutputStreamWriter(os, ENCODE);// 以UTF-8格式写入文件，不乱码
			out.write(content);
			out.close();
			os.close();
		} catch (Exception e) {
			e.printStackTrace();
		}
	}

	/**
	 * 读取本地文件到HDFS系统<br>
	 * 请保证文件格式一直是UTF-8，从本地->HDFS
	 */
	public void copyFileToHDFS(String localFile, String hdfsFile) {
		try {
			Path f = new Path(hdfsFile);
			File file = new File(localFile);

			FileInputStream is = new FileInputStream(file);
			InputStreamReader isr = new InputStreamReader(is, ENCODE);
			BufferedReader br = new BufferedReader(isr);

			FSDataOutputStream os = fs.create(f, true);
			Writer out = new OutputStreamWriter(os, ENCODE);

			String str;
			while ((str = br.readLine()) != null) {
				out.write(str + "\n");
			}
			br.close();
			isr.close();
			is.close();
			out.close();
			os.close();
			System.out.println("Write content of file " + file.getName() + " to hdfs file " + f.getName() + " success");
		} catch (Exception e) {
			e.printStackTrace();
		}
	}

	/**
	 * 取得文件块所在的位置..
	 */
	public void getLocation(String filePath) {
		try {
			Path f = new Path(filePath);
			FileStatus fileStatus = fs.getFileStatus(f);

			BlockLocation[] blkLocations = fs.getFileBlockLocations(fileStatus, 0, fileStatus.getLen());
			for (BlockLocation currentLocation : blkLocations) {
				String[] hosts = currentLocation.getHosts();
				for (String host : hosts) {
					System.out.println(host);
				}
			}
			// 取得最后修改时间
			long modifyTime = fileStatus.getModificationTime();
			Date d = new Date(modifyTime);
			System.out.println("取得最后修改时间:\n" + d);
		} catch (Exception e) {
			e.printStackTrace();
		}
	}

	// 文件重命名
	public void rename(String oldName, String newName) {
		Path oldPath = new Path(oldName);
		Path newPath = new Path(newName);
		boolean isok = false;
		try {
			isok = hdfs.rename(oldPath, newPath);
		} catch (IOException e) {
			e.printStackTrace();
		}
		if (isok) {
			System.out.println("rename ok!");
		} else {
			System.out.println("rename failure");
		}
	}

	// 创建目录
	public void mkdir(String path) {
		Path srcPath = new Path(path);
		boolean isok = false;
		try {
			isok = hdfs.mkdirs(srcPath);
		} catch (IOException e) {
			// TODO Auto-generated catch block
			e.printStackTrace();
		}
		if (isok) {
			System.out.println("create dir ok!");
		} else {
			System.out.println("create dir failure");
		}
	}

	/**
	 * 读取hdfs中的文件内容
	 */
	public void readFileFromHdfs(String filePath) {
		try {
			Path f = new Path(filePath);

			FSDataInputStream dis = fs.open(f);
			InputStreamReader isr = new InputStreamReader(dis, ENCODE);
			BufferedReader br = new BufferedReader(isr);
			String str;
			while ((str = br.readLine()) != null) {
				System.out.println(str);
			}
			br.close();
			isr.close();
			dis.close();
		} catch (Exception e) {
			e.printStackTrace();
		}
	}

	/**
	 * list all file/directory
	 * 
	 * @param args
	 * @throws IOException
	 * @throws IllegalArgumentException
	 * @throws FileNotFoundException
	 */
	public void listFileStatus(String path) throws FileNotFoundException, IllegalArgumentException, IOException {
		FileStatus fileStatus[] = fs.listStatus(new Path(path));
		int listlength = fileStatus.length;
		for (int i = 0; i < listlength; i++) {
			if (fileStatus[i].isDirectory() == false) {
				System.out
						.println("filename:" + fileStatus[i].getPath().getName() + "\tsize:" + fileStatus[i].getLen());
			} else {
				String newpath = fileStatus[i].getPath().toString();
				listFileStatus(newpath);
			}
		}
	}

	public static void main(String[] args) {
		// default dir : /user/root
		String createdFile = "/user/hadoop/test1";
		String content = "测试上传文件";
		String localFile = "/root/Downloads/hdfs编程.md";
		String hdfsFile = "/user/hadoop/fromLocal";
		String dirName = "/user/hadoop/dir1";
		String oldName = hdfsFile;
		String newName = "/user/hadoop/fromLocal_newname";
		//
		
		HadoopFSOperations a = new HadoopFSOperations();
		a.listDataNodeInfo();
		a.createFile(createdFile, content);
		a.checkFileExist(createdFile);
		a.copyFileToHDFS(localFile, hdfsFile);
		a.getLocation(createdFile);
		a.readFileFromHdfs(createdFile);
		a.deleteFile(newName);
		a.mkdir(dirName);
		a.rename(oldName, newName);
		
		try {
			a.listFileStatus(HADOOP_URL + "/");
			hdfs.close();
		} catch (FileNotFoundException e) {
			e.printStackTrace();
		} catch (IllegalArgumentException e) {
			e.printStackTrace();
		} catch (IOException e) {
			e.printStackTrace();
		}
	}
} 
```
注意灵活修改自己的`HADOOP_URL`。
我这台机器是`ubuntu 12.04` 
`hosts`文件里把127.0.1.1那行删掉改成自己的ip和机器名。
而且有运行hdfs。所以一切安好,运行后木有warning。

- 设置hdfs每个数据卷的保留空间：`hdfs-site.xml`
```xml
<property>
  <name>dfs.datanode.du.reserved</name>
  <value>6000000000</value>
  <description>Reserved space in bytes per volume. Always leave this much space free for non dfs use.
  </description>
</property>
```
