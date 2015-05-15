---
title: "BuildKVMImages"
date: 2015-05-11 15:33:37
tags: [kvm,libvirt]
categories: [Tech]
---





## 手动创建KVM虚拟机镜像

通过libvirt系的本地命令行管理工具，也可以方便地创建虚拟机。

* qemu-img：生成虚拟机磁盘文件
* virsh：命令行虚拟机管理工具

### 生成Domain XML文件

如果要用X11远程到物理机使用GUI工具，需要配置sshd，安装`xauth`：

`/etc/ssh/sshd_config`:

```
...
X11Forwarding yes
X11UseLocalhost no
...
```

```bash
$ /etc/init.d/sshd reload
$ yum install -y xauth
$ ssh -X user@host
$ virt-manager&
```

也可以直接用官方的example修改配置：

```
<domain type = 'kvm'>
        <name>centos66</name>
        <memory>1048576</memory>
        <vcpu>1</vcpu>
        <os>
                <type arch = 'x86_64'machine = 'pc'>hvm</type>
                <boot dev = 'cdrom'/>
                <boot dev = 'hd'/>
        </os>
        <features>
                <acpi/>
                <apic/>
                <pae/>
        </features>
        <clock offset = 'utc'/>
        <on_poweroff>destroy</on_poweroff>
        <on_reboot>restart</on_reboot>
        <on_crash>destroy</on_crash>
        <devices>
                <emulator>/usr/libexec/qemu-kvm</emulator>
                <disk type = 'file' device = 'disk'>
                        <driver name = 'qemu' type = 'raw' cache='none'/>
                        <source file = '/opt/vmdisks/centos66.raw'/>
                        <target dev='vda' bus='virtio'/>
                </disk>
                <disk type = 'file' device = 'cdrom'>
                        <source file = '/opt/CentOS-6.6-x86_64-bin-DVD1.iso'/>
                        <target dev = 'hdb' bus = 'ide'/>
                </disk>
"centos66.xml" 48L, 1243C written                                                                     8,20-34       Top
                        <target dev = 'hdb' bus = 'ide'/>
                </disk>
                <interface type='network'>
                        <source network='default'/>
                        <model type='virtio'/>
                </interface>
                <input type='tablet' bus='usb'/>
                <input type='mouse' bus='ps2'/>
                <graphics type='vnc' port='-1' autoport='yes' listen='0.0.0.0'>
                        <listen type='address' address='0.0.0.0'/>
                </graphics>
        </devices>
</domain>
```


### 创建虚拟机

```bash
$ mkdir -p /opt/vmdisks
$ qemu-img create -f raw /opt/vmdisks/centos66.raw 8G
$ virsh create centos66.xml
$ ip addr
$ virsh vncdisplay centos66 #获取ip和vnc端口登陆
```

接下来就进入安装操作系统的界面，一路按指印即可。

### 远程管理libvirt主机

可以用qemu+ssh，节点间互信后迁移；也可以用qemu+tcp，修改`/etc/libvirt/libvirtd.conf`:

```
listen_tls = 0
listen_tcp = 1
tcp_port = "16509"
listen_addr = "0.0.0.0"
auth_tcp = "none" # 生产环境建议加上授权验证
```

修改`/etc/init.d/libvirtd`:

```
LIBVIRTD_CONFIG=/etc/libvirt/libvirtd.conf
LIBVIRTD_ARGS="--listen"
```

```bash
$ service libvirtd restart
$ ss -ln  | grep 16509
```

在其他节点就可以连接（注意iptables）：

```bash
$ virsh -c qemu+tcp://192.168.182.156/system
```

克隆虚拟机：

```bash
$ virsh define centos66.xml
$ virt-clone --original=centos66 --name=centos66_01 -f centos66_01.raw
$ virsh list --all
```


### 参考

1. [libvirt官方虚拟机规格XML格式说明][libvirt0]


[libvirt0]:http://libvirt.org/formatdomain.html "libvirt官方虚拟机规格XML格式说明"