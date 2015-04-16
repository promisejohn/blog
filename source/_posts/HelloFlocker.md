---
title: "Hello Flocker"
date: 2015-04-16 23:56:04
tags: [flocker, docker, vagrant]
categories: [虚拟化]

---

### Flocker安装部署
安装flocker-deploy：

```bash
$ pyenv virtualenv flocker
$ pyenv activate flocker
$ pip install https://storage.googleapis.com/archive.clusterhq.com/downloads/flocker/Flocker-0.3.2-py2-none-any.whl
$ which flocker-deploy # 验证flocker-deploy安装在$(dirname `pyenv which python`)目录下
```
安装flocker-node，本地安装方式官方目前只支持fedora 20的linux，暂时选择用最简单的vagrant来测试。

准备第三方库：

```bash
$ cd /etc/yum.repos.d
$ wget http://download.virtualbox.org/virtualbox/rpm/$ rhel/virtualbox.repo # 导入virtualbox的yum库，修改默认不启用：enabled=0
$ wget http://mirrors.zju.edu.cn/epel/6/i386/epel-release-6-8.noarch.rpm
$ rpm -ivh epel-release-6-8.noarch.rpm # 导入EPEL库
```

安装VirtualBox：

```bash
$ cd ~/dev # 工作目录
$ yum install dkms ＃ 从EPEL安装
$ yum groupinstall "Development Tools" 
$ yum install kernel-devel # 确保当前运行的uname -r与kernel-devel版本一致
$ yum --enablerepo virtualbox install VirtualBox-4.3
$ /etc/init.d/vboxdrv setup
$ wget http://dlc-cdn.sun.com/virtualbox/4.3.26/Oracle_VM_VirtualBox_Extension_Pack-4.3.26-98988.vbox-extpack
$ VBoxManage extpack install Oracle_VM_VirtualBox_Extension_Pack-4.3.26-98988.vbox-extpack
```

安装Vagrant：

```bash
$ wget https://dl.bintray.com/mitchellh/vagrant/vagrant_1.7.2_x86_64.rpm
$ rpm -ivh vagrant_1.7.2_x86_64.rpm
$ cd ~/dev && mkdir HelloFlocker && cd $_
$ wget http://flocker.readthedocs.org/en/latest/_downloads/Vagrantfile
$ vagrant up
$ vagrant status
```
Flocker使用SSH通道：

```bash
$ eval $(ssh-agent) # 启用一个SSH Agent，可以用ssh-add测试是否启用
$ ssh-add ~/.vagrant.d/insecure_private_key
$ ssh root@172.16.255.250 flocker-reportstate --version # 检查virtualbox中虚拟机内的flocker-node版本
```

Flocker部署mongodb:
配置`minimal-application.yml`:

```yaml
"version": 1
"applications":
  "mongodb-example":
    "image": "clusterhq/mongodb"
```
配置`minimal-deployment.yml`:

```yaml
"version": 1
"nodes":
  "172.16.255.250": ["mongodb-example"]
  "172.16.255.251": []
```

检查docker服务端容器启用情况：

```bash
$ ssh root@172.16.255.250 docker ps
$ ssh root@172.16.255.251 docker ps
```

通过Flocker-deploy部署应用：

```bash
$ pyenv activate flocker # 确保进入隔离python环境，之前安装的flocker-cli在该环境内
$ flocker-deploy minimal-deployment.yml minimal-application.yml
$ ssh root@172.16.255.250 docker ps # 可以看到mongodb容器已启动
$ ssh root@172.16.255.251 docker ps
```

通过Flocker-deploy迁移应用：
编辑`minimal-deployment.yml`，可另存为`minimal-deployment-moved.yml`：

```yaml
"version": 1
"nodes":
  "172.16.255.250": []
  "172.16.255.251": ["mongodb-example"]
```
执行迁移：

```bash
$ flocker-deploy minimal-deployment-moved.yml minimal-application.yml
$ ssh root@172.16.255.250 docker ps # 看到容器列表空
$ ssh root@172.16.255.251 docker ps # 看到容器启动
```

Todo: 验证迁移过程中关联应用的影响，如持续读写mongodb时迁移。


参考：
1. [Flocker Doc][flocker1]
2. [Web开发环境搭建][flocker2]
3. [CentOS下安装VirtualBox][flocker3]
4. [CentOS6.5菜鸟之旅：安装VirtualBox4.3][flocker4]

[flocker1]: http://flocker.readthedocs.org/en/latest/gettingstarted/installation.html "Flocker doc on readthedoc.org"
[flocker2]: http://promisejohn.github.io/2015/04/15/WebDevEnvSetting/ "Web开发环境搭建"
[flocker3]: http://wiki.centos.org/zh/HowTos/Virtualization/VirtualBox "CentOS下安装VirtualBox"
[flocker4]: http://www.cnblogs.com/fsjohnhuang/p/3976331.html "CentOS6.5菜鸟之旅：安装VirtualBox4.3"