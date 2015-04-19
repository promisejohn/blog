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
$ wget http://flocker.readthedocs.org/en/latest/_downloads/Vagrantfile # See Below
$ vagrant up
$ vagrant status
```
使用官方的flocker box：

```ruby
Vagrant.require_version ">= 1.6.2"
VAGRANTFILE_API_VERSION = "2"
ENV['VAGRANT_DEFAULT_PROVIDER'] = 'virtualbox'
Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|
  config.vm.box = "clusterhq/flocker-tutorial"
  config.vm.box_version = "= 0.3.2"
  if Vagrant.has_plugin?("vagrant-cachier")
    config.cache.scope = :box
  end
  config.vm.define "node1" do |node1|
    node1.vm.network :private_network, :ip => "172.16.255.250"
    node1.vm.hostname = "node1"
  end
  config.vm.define "node2" do |node2|
    node2.vm.network :private_network, :ip => "172.16.255.251"
    node2.vm.hostname = "node2"
  end
end
```

Flocker使用SSH通道：

```bash
$ eval $(ssh-agent) # 启用一个SSH Agent，可以用ssh-add测试是否启用
$ ssh-add ~/.vagrant.d/insecure_private_key
$ ssh -t root@172.16.255.250 flocker-reportstate --version # 检查virtualbox中虚拟机内的flocker-node版本
```
在虚拟机中下载docker比如官方的MySQL案例：

```bash
$ ssh -t root@172.16.255.250 docker pull mysql:5.6.17
$ ssh -t root@172.16.255.251 docker pull mysql:5.6.17
```

### Flocker部署MySQL:
配置`mysql-application.yml`:

```yaml
"version": 1
"applications":
  "mysql-volume-example":
    "image": "mysql:5.6.17"
    "environment":
      "MYSQL_ROOT_PASSWORD": "clusterhq"
    "ports":
    - "internal": 3306
      "external": 3306
    "volume":
      "mountpoint": "/var/lib/mysql"
```
配置`mysql-deployment.yml`:

```yaml
"version": 1
"nodes":
  "172.16.255.250": ["mysql-volume-example"]
  "172.16.255.251": []
```

检查docker服务端镜像及容器启用情况：

```bash
$ ssh -t root@172.16.255.250 docker images
$ ssh -t root@172.16.255.250 docker ps
$ ssh -t root@172.16.255.251 docker images
$ ssh -t root@172.16.255.251 docker ps
```

通过Flocker-deploy部署应用：

```bash
$ pyenv activate flocker # 确保进入隔离python环境，之前安装的flocker-cli在该环境内
$ flocker-deploy mysql-deployment.yml mysql-application.yml
$ ssh root@172.16.255.250 docker ps # 可以看到MySQL容器已启动
$ ssh root@172.16.255.251 docker ps
```
使用MySQL服务：

```bash
$ mysql -uroot -pclusterhq -h172.16.255.250
$ # create some databases, tables and data.
$ # stay in the mysql client and keep connection.
```


### 通过Flocker-deploy迁移应用：
新增`mysql-deployment-moved.yml`：

```yaml
"version": 1
"nodes":
  "172.16.255.250": []
  "172.16.255.251": ["mysql-volume-example"]
```
执行迁移：

```bash
$ flocker-deploy mysql-deployment-moved.yml mysql-application.yml
$ ssh -t root@172.16.255.250 docker ps # 看到容器列表空
$ ssh -t root@172.16.255.251 docker ps # 看到容器启动
```
迁移过程中可以发现，mysql客户端连接的172.16.255.250服务端并没有断开。通过登录虚拟机可以找到原因（有兴趣的可以用tcpdump、ss来追踪）：

```bash
$ ssh root@172.16.255.250
$ root@250: iptables -t nat -L
```
可以看到flocker是通过Linux Netfilter的NAT表进行了默认的转发：

```
Chain PREROUTING (policy ACCEPT)
target     prot opt source               destination
DOCKER     all  --  anywhere             anywhere             ADDRTYPE match dst-type LOCAL
DNAT       tcp  --  anywhere             anywhere             tcp dpt:mysql ADDRTYPE match dst-type LOCAL /* flocker create_proxy_to */ to:172.16.255.251

Chain INPUT (policy ACCEPT)
target     prot opt source               destination

Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination
DOCKER     all  --  anywhere            !loopback/8           ADDRTYPE match dst-type LOCAL
DNAT       tcp  --  anywhere             anywhere             tcp dpt:mysql ADDRTYPE match dst-type LOCAL to:172.16.255.251

Chain POSTROUTING (policy ACCEPT)
target     prot opt source               destination
MASQUERADE  all  --  172.17.0.0/16        anywhere
MASQUERADE  tcp  --  anywhere             anywhere             tcp dpt:mysql
```
由此实现了应用的无缝迁移。

### 小结
Flocker实现的功能很有诱惑力，让docker具备了自动化的应用迁移能力，他俩关系有点像openstack和hypervisor的关系。但是，它目前还非常不成熟（0.4版在开发中），经常会遇到一些bug，就连官方也不推荐在生产环境使用，继续观望。


参考：
1. [Flocker Doc][flocker1]
2. [Web开发环境搭建][flocker2]
3. [CentOS下安装VirtualBox][flocker3]
4. [CentOS6.5菜鸟之旅：安装VirtualBox4.3][flocker4]
5. 在验证应用无缝迁移过程中用到的工具：ss、tcpdump。

[flocker1]: http://flocker.readthedocs.org/en/latest/gettingstarted/installation.html "Flocker doc on readthedoc.org"
[flocker2]: http://promisejohn.github.io/2015/04/15/WebDevEnvSetting/ "Web开发环境搭建"
[flocker3]: http://wiki.centos.org/zh/HowTos/Virtualization/VirtualBox "CentOS下安装VirtualBox"
[flocker4]: http://www.cnblogs.com/fsjohnhuang/p/3976331.html "CentOS6.5菜鸟之旅：安装VirtualBox4.3"