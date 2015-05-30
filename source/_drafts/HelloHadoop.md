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


## 通过Docker部署集群
可以自己做docker image，简单起见可以先用sequenceiq的image。

单节点测试：

```bash
$ docker pull sequenceiq/hadoop-docker:2.7.0
$ docker run -it sequenceiq/hadoop-docker:2.7.0 /etc/bootstrap.sh -bash
$ cd $HADOOP_PREFIX # /usr/local/hadoop
$ bin/hadoop jar share/hadoop/mapreduce/hadoop-mapreduce-examples-2.7.0.jar grep input output 'dfs[a-z.]+' # 统计key出现次数
$ bin/hdfs dfs -cat output/* # 在用户Home下的output下
```

多节点部署，可以用ambari镜像，准备好blueprint.json：

```
{ "host_groups" : [
    { "name" : "host_group_1",
      "components" : [
        { "name" : "ZOOKEEPER_SERVER" },
        { "name" : "ZOOKEEPER_CLIENT" },
        { "name" : "AMBARI_SERVER" },
        { "name" : "HDFS_CLIENT" },
        { "name" : "NODEMANAGER" },
        { "name" : "MAPREDUCE2_CLIENT" },
        { "name" : "APP_TIMELINE_SERVER" },
        { "name" : "DATANODE" },
        { "name" : "YARN_CLIENT" },
        { "name" : "RESOURCEMANAGER" } ],
      "cardinality" : "1" },
    { "name" : "host_group_2",
      "components" : [
        { "name" : "ZOOKEEPER_SERVER" },
        { "name" : "ZOOKEEPER_CLIENT" },
        { "name" : "SECONDARY_NAMENODE" },
        { "name" : "NODEMANAGER" },
        { "name" : "YARN_CLIENT" },
        { "name" : "DATANODE" }],
      "cardinality" : "1" },
    { "name" : "host_group_3",
      "components" : [
        { "name" : "ZOOKEEPER_SERVER" },
        { "name" : "ZOOKEEPER_CLIENT" },
        { "name" : "NAMENODE" },
        { "name" : "NODEMANAGER" },
        { "name" : "YARN_CLIENT" },
        { "name" : "DATANODE" }],
      "cardinality" : "1" } ],
  "Blueprints" : {
    "blueprint_name" : "blueprint-c1",
    "stack_name" : "HDP",
    "stack_version" : "2.2" } }
```
可以在ambari server上查找：`http://172.17.0.13:8080/api/v1/blueprints`。

创建集群：

```bash
$ docker pull sequenceiq/ambari:1.7.0
$ curl -Lo .amb https://github.com/sequenceiq/docker-ambari/raw/master/ambari-functions && . .amb
$ amb-start-cluster 3 # 注意默认使用的是sequenceiq/ambari:1.7.0-warmup，可以修改 .amb
$ amb-shell # 又启了个container
ambshell > host list
ambshell > blueprint add --url http://172.17.42.1/bp.json # 准备好json保存到某个能访问的http服务器
ambshell > cluster build --blueprint blueprint-c1
ambshell > cluster assign --hostGroup host_group_1 --host amb0.mycorp.kom
ambshell > cluster assign --hostGroup host_group_2 --host amb1.mycorp.kom
ambshell > cluster assign --hostGroup host_group_3 --host amb2.mycorp.kom
ambshell > blueprint show --id blueprint-c1
ambshell > cluster preview
ambshell > cluster create
```
中间安装过程如果出现失败，可以到ambari-server上看详细情况，当然也可以直接向之前方式一样，通过GUI安装部署；或者开着浏览器看amb-shell执行过程。
如果实在VM上远程使用，可以在docker所在机器上做NAT映射直接访问：

```bash
$ iptables -t nat -A PREROUTING -d 10.101.29.26 -p tcp --dport 8000 -j DNAT --to-destination 172.17.0.20:8080 # 可以用端口转发直接远程浏览器访问
```

做个简单的测试：

```bash
$ docker exec -it 9572938bb253 /bin/bash # 进入到容器内
bash # su hdfs # 切换到HDFS的超级用户
bash # hdfs dfsadmin -report
bash # hdfs dfs -ls /
```


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
$ cd hellohdfs && mvn package
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
      <version>2.6.0</version>
    </dependency>
    <dependency>
      <groupId>org.apache.hadoop</groupId>
      <artifactId>hadoop-hdfs</artifactId>
      <version>2.6.0</version>
    </dependency>
    <dependency>
      <groupId>org.apache.hadoop</groupId>
      <artifactId>hadoop-client</artifactId>
      <version>2.6.0</version>
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
$ mkdir -p src/main/resources
$ scp cloudlab119:/etc/hadoop/conf/core-site.xml src/main/resources
$ mvn exec:java -Dexec.mainClass="org.tecstack.App" -Dexec.cleanupDaemonThreads=false
$ mvn dependency:copy-dependencies # 导出依赖的包
$ mvn resources:resources
$ 
```

关于maven的exec插件，也可以通过配置`pom.xml`的plugin的参数简化执行：

```xml
<build>
  <plugins>
    <plugin>
      <groupId>org.codehaus.mojo</groupId>
      <artifactId>exec-maven-plugin</artifactId>
      <version>1.4.0</version>
      <executions>
        <execution>
          <goals>
             <goal>java</goal>
          </goals>
        </execution>
      </executions>
      <configuration>
        <mainClass>org.tecstack.App</mainClass>
        <cleanupDaemonThreads>false</cleanupDaemonThreads>
      </configuration>
   </plugin>
 </plugins>
</build>
```

之后运行只需要`mvn exec:java`，如果需要定制更多参数，比如JVM内存，单独启动进程执行等，可以使用`exec:exec`插件，具体看[codehaus官网][http://mojo.codehaus.org/exec-maven-plugin/usage.html]。


## HUE交互界面

Hue是一个基于Django开发的webapp，通过WebHDFS或HttpFS的一种访问HDFS的数据，在HDFS HA部署方式中只能使用HttpFS。
如通过WebHDFS方式访问，需要修改`hdfs-site.xml`：

```xml
<property>
  <name>dfs.webhdfs.enabled</name>
  <value>true</value>
</property>
```
修改`core-site.xml`使Hue账号可以代理其他用户：

```xml
<property>
  <name>hadoop.proxyuser.hue.hosts</name>
  <value>*</value>
</property>
<property>
  <name>hadoop.proxyuser.hue.groups</name>
  <value>*</value>
</property>
```
修改完参数需要重新启动HDFS，通过Ambari操作很方便。

```bash
$ yum install -y hue hue-server
$ vim /etc/hue/conf/hue.ini # 修改各个服务的URI，可以通过Ambari看到服务所在的服务器位置；修改默认监听端口不与其它冲突
$ service hue start
```

默认登陆账号为`admin:admin`，可以通过hdfs为该账号创建一个专用的文件夹`/user/hue`进行测试，正式环境中可以考虑与其他身份系统对接。

```bash
$ su hdfs
$ hdfs dfs -mkdir /user/hue
$ hdfs dfs -chown -R admin:hadoop  /user/hue
$ hdfs dfs -ls /user/
```

## 导入数据到HBase

通过hcat建立数据库表结构，通过pig导入数据。

建立数据文件`data.tsv`:

```
row1    c1    c2
row2    c1    c2
row3    c1    c2
row4    c1    c2
row5    c1    c2
row6    c1    c2
row7    c1    c2
row8    c1    c2
row9    c1    c2
row10    c1    c2
```

准备建表DDL，`simple.ddl`：

```
CREATE TABLE simple_hcat_load_table (id STRING, c1 STRING, c2 STRING)
STORED BY 'org.apache.hadoop.hive.hbase.HBaseStorageHandler'
WITH SERDEPROPERTIES ( 'hbase.columns.mapping' = 'd:c1,d:c2' )
TBLPROPERTIES (
'hbase.table.name' = 'simple_hcat_load_table'
); 
```

准备导入脚本，`simple.bulkload.pig`：

```
A = LOAD 'hdfs:///tmp/data.tsv' USING PigStorage('\t') AS (id:chararray, c1:chararray, c2:chararray);
-- DUMP A;
STORE A INTO 'hbase://simple_hcat_load_table' USING org.apache.hive.hcatalog.pig.HCatStorer();
```

把文件放到HDFS，建表，导入：

```bash
$ su - hdfs
$ hdfs dfs -put data.tsv /tmp/
$ hcat -f simple.ddl
$ pig -useHCatalog simple.bulkload.pig
```

通过Hue的HCat可以查看数据库和表数据，也可以通过HBase Shell：

```bash
$ su - hdfs
$ hbase shell
hbase > list # 查询所有表
hbase > scan 'simple_hcat_load_table' # 查询所有数据
hbase > describe 'simple_hcat_load_table'
```

也可以通过Hive Shell：

```bash
$ su - hdfs
$ hive
hive > show tables;
hive > select count(*) from simple_hcat_load_table;
hive > desc simple_hcat_load_table;
```


如果建表过程中出现类似`java.lang.NoClassDefFoundError: org/apache/hadoop/hbase/HBaseConfiguration`如下的找不到类情况，检查hcat的配置文件是否有HBase目录：`vim /usr/bin/hcat`。

```bash
...
export HBASE_HOME=/usr/hdp/2.2.4.2-2/hbase
...
```
此外，由于Hue集成的HCat调用的是Hive下的hcat，需要在`/etc/hive-hcatalog/conf/hcat-env.sh`中指定`HBASE_HOME`:

```bash
...
HBASE_HOME=${HBASE_HOME:-/usr/hdp/current/hbase-client}
...
```

pig在使用MapReduce模式执行时，可以根据log打开MR Job跟踪，如果也出现`java.lang.NoClassDefFoundError`类错误，说明mapreduce服务的classpath有缺漏，可以通过Ambari修改MapReduce服务`Advanced mapred-site`配置中的`mapreduce.application.classpath`，比如出现HBase相关类找不到，则添加HBase相关的库`/usr/hdp/2.2.4.2-2/hbase/lib/*`，然后重启MapReduce服务即可。


如果出现执行权限错误，需要检查如下配置是否存在：
`hive-site.xml`:

```xml
<property>
  <name>hive.security.metastore.authorization.manager</name>
  <value>org.apache.hadoop.hive.ql.security.authorization.StorageBasedAuthorizationProvider
</value>
</property>
<property>
  <name>hive.security.metastore.authenticator.manager</name>
  <value>org.apache.hadoop.hive.ql.security.HadoopDefaultMetastoreAuthenticator
</value>
</property>
<property>
  <name>hive.metastore.pre.event.listeners</name>
  <value>org.apache.hadoop.hive.ql.security.authorization.AuthorizationPreEventListener
</value>
</property>
<property>
  <name>hive.metastore.execute.setugi</name>
  <value>true</value>
</property>
```

`webhcat-site.xml`:
正式环境中可以明确具体使用的账号名称，最小化权限。

```xml
<property>
  <name>webhcat.proxyuser.hue.hosts</name>
  <value>*</value>
</property>
<property>
  <name>webhcat.proxyuser.hue.groups</name>
  <value>*</value>
</property>
```

`core-site.xml`:

```xml
<property>
  <name>hadoop.proxyuser.hcat.group</name>
  <value>*</value>
</property>
<property>
  <name>hadoop.proxyuser.hcat.hosts</name>
  <value>*</value>
</property>
```


## 使用小结

Hadoop系架构经过引入YARN，真正实现了数据和应用的分离，在YARN架构上可以实现出了原来的MapReduce之外的更多App，比如流式处理、实时处理、图计算等，如此可以真正实现“把应用挪到数据旁边高效执行”，构建大数据平台。此外，做数据分析时可以发现周边工具很丰富，比如hue统一的综合管理界面、从数据文件识别结构并管理数据的hcat、数据处理高级语言pig、支持SQL的Hive，使得在Hadoop平台上做数据分析非常方便（可以参考[这里][hadoop10]）。
最后，数据处理优选python scikit-learn系工具，当大到一定程度，或需要多人同时工作时，hadoop系平台是个不错的选择。


## 参考

1. [Ambari Official Docs][hadoop0]
2. [HDFS Official Doc][hadoop1]
3. [Hadoop-docker from sequenceiq][hadoop2]
4. [Ambari Docs on Apache][hadoop3]
5. [Sample BluePrint][hadoop4]
6. [安装配置Hue][hadoop5]
7. [手动安装HDP][hadoop6]
8. [Ambari Blueprint][hadoop7]
9. [导入数据到HBase][hadoop8]


[hadoop0]:http://docs.hortonworks.com/HDPDocuments/Ambari-2.0.0.0/Ambari_Doc_Suite/ADS_v200.html "Ambari Official Docs"
[hadoop1]:http://hadoop.apache.org/docs/stable2/hadoop-project-dist/hadoop-common/FileSystemShell.html "HDFS Official Doc"
[hadoop2]:https://github.com/sequenceiq/hadoop-docker "hadoop-docker ffrom sequenceiq"
[hadoop3]:https://cwiki.apache.org/confluence/display/AMBARI/Blueprints#Blueprints-Step1:CreateBlueprint "Ambari docs on apache"
[hadoop4]:https://blog.codecentric.de/en/2014/05/lambda-cluster-provisioning/ "Sample Blueprint"
[hadoop5]:http://ju.outofmemory.cn/entry/128881 "安装配置Hue"
[hadoop6]:http://docs.hortonworks.com/HDPDocuments/HDP2/HDP-2.2.4/HDP_Man_Install_v224/index.html "手动安装HDP"
[hadoop7]:https://cwiki.apache.org/confluence/display/AMBARI/Blueprints "Ambari Blueprint"
[hadoop8]:http://docs.hortonworks.com/HDPDocuments/HDP2/HDP-2.2.4/Importing_Data_HBase_v224/index.html "导入数据到HBase"
[hadoop9]:https://cwiki.apache.org/confluence/display/Hive/WebHCat+InstallWebHCat "WebHCat部署"
[hadoop10]:http://zh.hortonworks.com/hadoop-tutorial/hello-world-an-introduction-to-hadoop-hcatalog-hive-and-pig/ "hcat & hive & pig"