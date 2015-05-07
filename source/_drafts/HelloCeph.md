---
title: "HelloCeph"
tags: [ceph]
categories: [Tech]
---

## Cinder with Ceph

Ceph支持对象存储、块存储、分布式文件系统，所有集群都由至少1个Monitor和2个OSD组成，MDS（Metadata Server）在作为文件系统时必须存在。

* Monitor：集群状态管理
* OSD：数据的存储、恢复、复制、平衡等，节点间互相检测状态
* MDS：只有Ceph Filesystem用到，提供POSIX客户端接口

目前官方推荐的底层配置主要是ubuntu12.04，配合xfs或ext4文件系统。

### 安装ceph
测试条件下，在osceph0、osceph1、osceph2分别启用3个OSD，一共9个OSD，在osceph0上部署ceph monitor。

新增`/etc/yum.repos.d/ceph.repo`：

```
[ceph-noarch]
name=Ceph noarch packages
baseurl=http://ceph.com/rpm/el6/noarch
enabled=1
gpgcheck=0
type=rpm-md
gpgkey=https://ceph.com/git/?p=ceph.git;a=blob_plain;f=keys/release.asc
gpgcheck=0

[ceph-x86]
name=Ceph x86 packages
baseurl=http://ceph.com/rpm/el6/x86_64
enabled=1
```
安装`ceph-deploy`，该工具是ceph的管理工具：

```bash
$ yum install -y ceph-deploy
# ceph最近需要FQ，可以FQ后下载rpm包安装
$ wget http://ceph.com/rpm/el6/noarch/ceph-deploy-1.5.23-0.noarch.rpm
$ yum install ceph-deploy-1.5.23-0.noarch.rpm
```
`ceph-deploy`需要一个具备无密码sudo权限的账号访问每个node，此外需要从管理节点到所有节点做免密码登录认证，同时执行ceph-deploy的时候需要切换到非root用户权限，最后检查iptables和SELinux配置是否正确：

```bash
$ su - ceph
$ mkdir cephadmin && cd $_
$ for i in osceph{0,1,2}; do ssh root@$i useradd -d /home/ceph -m ceph;done;
# 密码'ceph'，生产环境不建议，这样有明文密码传输。
$ for i in osceph{0,1,2}; do ssh root@$i passwd ceph;done;
$ for i in osceph{0,1,2}; do ssh root@$i 'echo "ceph ALL = (root) NOPASSWD:ALL" | tee /etc/sudoers.d/ceph'; done;
$ for i in osceph{0,1,2}; do ssh root@$i chmod 0440 /etc/sudoers.d/ceph;done;
$ for i in osceph{0,1,2}; do echo ssh-keygen -t rsa -P '' -f cephadmin/$i.key;done;
$ for i in osceph{0,1,2}; do ssh-copy-id -i cephadmin/$i.key.pub ceph@$i;done;
```
编辑`~/.ssh/config`，注意修改key的位置：

```
Host osceph0
   Hostname osceph0
   User ceph
   PreferredAuthentications publickey
   IdentityFile ~/cephadmin/osceph0.key
Host osceph1
   Hostname osceph1
   User ceph
   PreferredAuthentications publickey
   IdentityFile ~/cephadmin/osceph1.key
Host osceph2
   Hostname osceph2
   User ceph
   PreferredAuthentications publickey
   IdentityFile ~/cephadmin/osceph2.key
```

### 部署节点
添加一个Monitor：

```bash
$ su - ceph
$ mkdir ~/cephconf && cd $_
$ ceph-deploy new osceph0
$ echo "osd pool default size = 2" >> ceph.conf
$ ceph-deploy install osceph0 osceph1 osceph2
$ ceph-deploy mon create-initial
```
添加2个OSD：

```bash
$ ssh osceph1 mkdir -p /opt/ceph/osd0
$ ssh osceph2 mkdir -p /opt/ceph/osd0
$ ceph-deploy osd prepare osceph1:/opt/ceph/osd0
$ ceph-deploy osd prepare osceph2:/opt/ceph/osd0
$ ceph-deploy osd activate osceph1:/opt/ceph/osd0
$ ceph-deploy osd activate osceph2:/opt/ceph/osd0
```

如果中间出问题，可以清除数据重新开始安装：

```bash
$ ceph-deploy purgedata osceph0 osceph1 osceph2
$ ceph-deploy forgetkeys
$ ceph-deploy purge osceph0 osceph1 osceph2
```

参考：

1. [Ceph Official Docs][ceph0]

[ceph0]:http://docs.ceph.com/docs "ceph official docs"