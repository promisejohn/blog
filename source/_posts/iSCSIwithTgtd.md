---
title: 使用Linux的tgtd提供iscsi服务
tags: [iscsi, tgtd, linux]
categories: [Tech]
date: 2015-06-14 15:20:05
---



## 安装部署

IP | 角色
---|----
192.168.182.156 |  tgtd
192.168.182.157 | client1
192.168.182.158 | client2


```bash
$ yum install -y scsi-target-utils
$ chkconfig tgtd on
$ service tgtd start
```

配置方法2种：

1. tgtadm，在线修改
2. conf配置文件

## 在服务端增加一个Target
主要流程：

1. 建立target
2. 为target增加backstorage
3. 配置客户端访问target的控制策略

```bash
$ tgtadm --lld iscsi --op new --mode target --tid 1 --targetname iqn.2015-05-04.org.tecstack.storage.tg1 # add target
# 用losetup映射块设备作为backstorage
$ mkdir /opt/tgtstorage
$ dd if=/dev/zero of=/opt/tgtstorage/disk0.img bs=1M count=5120
$ losetup -f /opt/tgtstorage/disk0.img # 映射为设备
$ losetup -a # /dev/loop0
$ tgtadm --lld iscsi --op new --mode logicalunit --tid 1 --lun 1 --backing-store /dev/loop0
# 直接用块文件作为backstorage添加第二个块
$ dd if=/dev/zero of=/opt/tgtstorage/disk1.img bs=1M count=5120
$ tgtadm --lld iscsi --op new --mode logicalunit --tid 1 --lun 2 --backing-store /opt/tgtstorage/disk1.img
# 绑定客户端ip
$ tgtadm --lld iscsi --mode target --op bind --tid 1 --initiator-address=192.168.182.157
$ tgtadm --lld iscsi --mode target --op bind --tid 1 --initiator-address=192.168.182.158
$ tgt-admin --dump |grep -v default-driver > /etc/tgt/conf.d/my-targets.conf # 通过tgt-admin保存为配置文件，注意与tgtadm的区别！
$ tgtadm --lld iscsi --mode target --op show
$ tgt-admin --show # 和上一个命令一样，tgt-admin是tgtadm的perl封装
```

在服务端禁用某个客户端：

```bash
$ tgtadm --lld iscsi --mode target --op unbind --tid 1 --initiator-address=192.168.182.158 # unbind，使不可发现
```

## 在客户端使用target
iscsi存储使用的主要流程：

1. 发现target
2. login到target
3. 使用存储管理工具使用块设备

```bash
$ iscsiadm --mode discovery --type sendtargets --portal 192.168.182.156
$ ls -lh /var/lib/iscsi/nodes/ # 可以查看到对应的target和卷资料
$ iscsiadm -m node # 查看当前机器上所有target
$ iscsiadm -m node -T iqn.2015-05-04.org.tecstack.storage.tg1 --login # 登陆target
$ fdisk -l # 登陆后就可以看到块设备
$ iscsiadm -m node -T iqn.2015-05-04.org.tecstack.storage.tg1 --logout #登出target
$ fdisk -l # 登出后设备移除
$ iscsiadm -m node -o delete -T iqn.2015-05-04.org.tecstack.storage.tg1 # 删除target
$ ls -lh /var/lib/iscsi/nodes/
$ iscsiadm -m node -T iqn.2015-05-04.org.tecstack.storage.tg1 -p 192.168.182.156 -- op update -n node.startup -v automatic # 自动login
```

当login到target之后，就可以使用，比如通过LVM：

```bash
$ fdisk -l # 增加了一个/dev/sdb块设备
$ pvcreate /dev/sdb
$ pvdisplay
$ vgcreate myiscsi /dev/sdb
$ vgdisplay
$ lvcreate -l 1024 -n vdisk0 myiscsi
$ lvdisplay
$ ls -lh /dev/myiscsi/vdisk0 # 生成的块设备位置
$ mkfs.ext4 /dev/myiscsi/vdisk0
$ mkdir -p /opt/myiscsidata
$ mount /dev/myiscsi/vdisk0 /opt/myiscsidata/
$ df -h
```

运行过程中为target添加后端存储，客户端需要logout后重新login才能看到。但是如果重新login后块设备名会变化，比如变成`/dev/sdc`。可以通过文件系统的UUID来识别设备并挂载：

```bash
$ tune2fs -l /dev/sdc
```

### GFS2测试

#### 创建和挂载iscsi块存储

在target端创建LUN

```bash
$ dd if=/dev/zero of=/opt/tgtstorage/disk_gfs.img bs=1M count=5120
$ tgtadm --lld iscsi --op new --mode logicalunit --tid 1 --lun 3 --backing-store /opt/tgtstorage/disk_gfs.img
$ tgtadm --lld iscsi --mode target --op bind --tid 1 --initiator-address=192.168.182.157
$ tgtadm --lld iscsi --mode target --op bind --tid 1 --initiator-address=192.168.182.158
$ tgt-admin --show
```

在client1和client2上挂载块设备：

```bash
$ iscsiadm --mode discovery --type sendtargets --portal 192.168.182.156
$ iscsiadm -m node
$ iscsiadm -m node -T iqn.2015-05-04.org.tecstack.storage.tg1 --login
$ fdisk -l
```

#### 使用luci配置集群：

在target端配置：

```bash
$ yum -y install luci
$ service luci start
```

在client1和client2上配置：

```bash
$ yum install -y ricci
$ service ricci start
$ passwd ricci # 123456，节点密码
```

通过访问target的https（默认端口8084，账号同Linux本地root账号），创建集群，新增节点，选择下载包，勾选启用共享文件系统。管理工具会自动帮助安装：`cman rgmanager lvm2-cluster sg3_utils gfs2-utils`，并启动相关服务。

#### 在一台集群节点上创建LVM逻辑卷，格式化为GFS文件系统

识别scsi和块设备的对应关系：

```bash
$ ls -l /dev/disk/by-path/*tecstack*
$ scsi_id -gu /dev/sdb
```

在client1上执行：

```bash
$ pvcreate /dev/sdb
$ vgcreate gfstest /dev/sdb
$ lvcreate -l 1024 -n gfsdisk0 gfstest
$ mkfs.gfs2 -j2 -p lock_dlm -t gfstest:gfs2 /dev/gfstest/gfsdisk0
```

#### 在两台集群节点上同时挂载GFS文件系统

在client1和client2上执行：

```bash
$ mkdir /mnt/gfstest
$ mount /dev/gfstest/gfsdisk0 /mnt/gfstest
```

参考：

1. [creating and managing iscsi targets][iscsi0]
2. [tgtadm man page][iscsi1]
3. [iscsi使用案例][iscsi2]
4. [GFS配置][iscsi3]

[iscsi0]: http://blog.delouw.ch/2013/07/07/creating-and-managing-iscsi-targets/ "creating and managing iscsi targets"
[iscsi1]: http://stgt.sourceforge.net/manpages/tgtadm.8.html "tgtadm official man page"
[iscsi2]: http://linux.vbird.org/linux_server/0460iscsi.php "iscsi使用案例"
[iscsi3]: https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux/5/html/Global_File_System/s1-config-tasks.html "GFS配置"
