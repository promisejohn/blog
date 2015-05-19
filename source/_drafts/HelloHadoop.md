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

```bash
$ su -s /bin/sh -c 'hadoop fs -ls /' hdfs
$ 
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


[hadoop0]:http://docs.hortonworks.com/HDPDocuments/Ambari-2.0.0.0/Ambari_Doc_Suite/ADS_v200.html "Ambari Official Docs"