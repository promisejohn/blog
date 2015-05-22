---
title: "HelloHadoop"
tags: []
categories: []
---


# Hadoop系列
大数据依旧在热炒，hadoop虽然不是唯一的代表，却也是各家必谈之资本，玩玩大数据。

## 通过ambari部署hadoop集群
在cloudlab119-123上部署集群，其中119作为ambari server控制端。

安装部署：

```bash
$ cd /etc/yum.repo.d
$ wget http://public-repo-1.hortonworks.com/ambari/centos6/2.x/updates/2.0.0/ambari.repo
$ yum install -y ambari-server
$ ambari-server setup # 按指示操作即可
$ ambari-server start # *:8080端口
```
配置，打开http://cloudlab119:8080，默认密码admin:admin。在cloudlab119上对所有节点做免密码认证：

```bash
$ vi /etc/hosts
$ ssh-keygen -t rsa -P '' -f ~/.ssh/hadoop119
$ for i in {119,120,121,122,123}; do ssh-copy-id -i ~/.ssh/hadoop119.pub root@cloudlab$i; done;
```
按提示建议操作，如ntpd、iptables、关闭THP：

```bash
$ chkconfig ntpd on
$ service ntpd start
$ service iptables stop # 生产环境中参照文档把对应端口打开
$ chkconfig iptables off
$ echo never > /sys/kernel/mm/redhat_transparent_hugepage/enabled # 关闭THP，如果提示文件系统readonly，重启机器再执行
```

按提示一步步操作即可，但中间如果出现下载包失败，则会导致整个安装失败，所以最好提前把官方源同步到本地，做镜像后安装。

```bash
$ yum install -y yum-utils createrepo
$ mkdir -p /var/www/html/ambari/centos6 && cd $_
$ reposync -r Updates-ambari-2.0.0
$ createrepo Updates-ambari-2.0.0
$ mkdir -p /var/www/html/hdp/centos6 && cd $_
$ reposync -r HDP-2.2
$ reposync -r HDP-UTILS-1.1.0.20
$ createrepo HDP-2.2
$ createrepo HDP-UTILS-1.1.0.20
$ vim /etc/yum.repo.d/ambari.repo # 修改baseurl指向本地镜像
```

安装之后，发现某几台服务器内存占用率非常高，可以通过Ambari增加Host节点，然后迁移部分的组件到新的机器。

## HDFS

HDFS由NameNode、SNameNode和DataNode组成，HA的时候还有另一个NameNode（StandBy）。SNameNode专门从NameNode获取FSImage和Edits，为NameNode合并生成新的FSImage。

HDFS有个超级用户，就是启动NameNode的那个linux账号。
### 基本Shell操作

```bash
$ ps -elf | grep NameNode #看看哪个账号启动了NameNode，那个账号就是超级用户
$ su -s /bin/sh -c 'hadoop fs -ls /' hdfs #使用超级账号执行命令
$ su hdfs #切换到超级账号
$ useradd -G hdfs promise
$ hdfs dfs -mkdir /user/promise
$ hdfs dfs -chown promise:hdfs /user/promise
$ hdfs dfs -ls /user/
$ hdfs dfs -mkdir -p /user/promise/dev/hello
$ hdfs dfs -chmod -R 777 /user/promise/dev
$ hdfs dfs -ls /user/promise/dev/
$ echo “helloworld” > hello.txt
$ hdfs dfs -put hello.txt /user/promise/helloworld.txt
$ hdfs dfs -put - hdfs://192.168.182.119/user/promise/test.txt # 从stdin输入，Ctrl-D结束
$ hdfs dfs -cat hdfs://192.168.182.119/user/promise/test.txt
$ hdfs dfs -cat /user/promise/helloworld.txt
$ hdfs dfs -get /user/promise/helloworld.txt hello2.txt
$ hdfs dfs -tail -f /user/promise/helloworld.txt # 查看追加的文件写入
$ hdfs dfs -appendToFile - /user/promise/helloworld.txt
```

### webhdfs操作

启用了webhdfs之后（`hdfs-site`中`dfs.webhdfs.enabled=true`），可以通过HTTP方式访问HDFS：

```bash
$ curl -i "http://192.168.182.119:50070/webhdfs/v1/user/promise/?op=LISTSTATUS"
$ curl -i -L "http://192.168.182.119:50070/webhdfs/v1/user/promise/helloworld.txt?op=OPEN" # 通过重定向到DataNode获取文件
$ curl -i -X PUT "http://192.168.182.119:50070/webhdfs/v1/user/promise/hello?user.name=promise&op=MKDIRS"
```
通过HTTP方式创建和追加文件都需要通过2阶段实现：先在NameNode上创建，获得DataNode URI后再向URI上传文件。

```bash
$ curl -i -X PUT "http://192.168.182.119:50070/webhdfs/v1/user/promise/hello/hi.txt?op=CREATE"
$ curl -i -X PUT -T hi.txt "http://192.168.182.119:50075/webhdfs/v1/user/promise/hello/hi.txt?user.name=promise&op=CREATE&namenoderpcaddress=192.168.182.119:8020&overwrite=false" # 需要声明账号
```

### 查看离线的FSImage和Edits：

```bash
$ hdfs oiv -p XML -i /hadoop/hdfs/namenode/current/fsimage_0000000000000018887 -o fsimage.xml # 生成XML格式
$ hdfs oiv  -i /hadoop/hdfs/namenode/current/fsimage_0000000000000018887 # 在线查看形式
$ hdfs dfs -ls webhdfs://127.0.0.1:5978/
$ hdfs oev -i /hadoop/hdfs/namenode/current/edits_0000000000000000001-0000000000000005150 -o edits.xml # 查看edits
```

### 使用Java API

Java在大型软件系统开发中有利于更清晰的架构设计和团队分工，而且有大量的第三方优质框架可用；可惜在命令里写HelloWorld很啰嗦，方便起见，直接用maven生成基本文件结构。

```bash
$ export PATH=$PATH:/opt/apache-maven-3.3.3/bin/
$ export JAVA_HOME=/usr/jdk64/jdk1.7.0_67/
$ export MAVEN_OPTS="-Xms256m -Xmx512m"
$ mkdir ~/dev && cd $_
$ mvn archetype:generate -DgroupId=org.tecstack -DartifactId=hellohdfs -DarchetypeArtifactId=maven-archetype-quickstart -DinteractiveMode=false
$ cd hellohdfs
$ vim ./src/main/java/org/tecstack/App.java
```

```java
package org.tecstack;

import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.fs.FileStatus;
import org.apache.hadoop.fs.FileSystem;
import org.apache.hadoop.fs.Path;

/**
 * Hello HDFS
 *
 */
public class App
{
    public static void main( String[] args )
    {
        try {
                Configuration conf=new Configuration();
                FileSystem hdfs=FileSystem.get(conf);
                Path dst =new Path("/user/promise/helloworld.txt");
                FileStatus files[]=hdfs.listStatus(dst);
                for(FileStatus file:files) {
                    System.out.println(file.getPath());
                }
        } catch (Exception e) {
                e.printStackTrace();
        }
    }
}
```

在`pom.xml`增加hadoop依赖包:

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/maven-v4_0_0.xsd">
  <modelVersion>4.0.0</modelVersion>
  <groupId>org.tecstack</groupId>
  <artifactId>hellohdfs</artifactId>
  <packaging>jar</packaging>
  <version>1.0-SNAPSHOT</version>
  <name>hellohdfs</name>
  <url>http://maven.apache.org</url>
  <dependencies>
    <dependency>
      <groupId>org.apache.hadoop</groupId>
      <artifactId>hadoop-common</artifactId>
      <version>2.2.0</version>
    </dependency>
    <dependency>
      <groupId>junit</groupId>
      <artifactId>junit</artifactId>
      <version>3.8.1</version>
      <scope>test</scope>
    </dependency>
  </dependencies>
</project>
```

打包运行：

```bash
$ mvn package
$ mvn exec:java -Dexec.mainClass="org.tecstack.App"
$ mvn dependency:copy-dependencies # 导出依赖的包
```



## Zookeeper

## 尝试HBase

## 尝试Hive

## 尝试Mahout
如果是相对数据初步分析从而得出部分经验模型，更喜欢python的scikit系工具，pandas、scipy、numpy……

## Spark

## Storm


## 参考

1. [Ambari Official Docs][hadoop0]
2. [HDFS Official Doc][hadoop1]


[hadoop0]:http://docs.hortonworks.com/HDPDocuments/Ambari-2.0.0.0/Ambari_Doc_Suite/ADS_v200.html "Ambari Official Docs"
[hadoop1]:http://hadoop.apache.org/docs/stable2/hadoop-project-dist/hadoop-common/FileSystemShell.html "HDFS Official Doc"