---
title: "Openstack安装部署"
tags: [Tech]
categories: [Tech]
---

## 参考架构及部署规划
![参考架构](http://docs.openstack.org/icehouse/install-guide/install/yum/content/figures/1/a/common/figures/openstack_havana_conceptual_arch.png)

* 操作系统：CentOS6.5
* 第三方yum源：epel, rdo

节点部署角色：

节点名 | internal ip | public ip | Role
----- | ------------| ----------| ----
oscontroller | 10.0.100.145 | 192.168.182.150 | nova, glance, cinder, image, neutron, dashboard
osnetwork | 10.0.100.146 | 192.168.182.151 | ML2, OVS, L2 Agent, L3 Agent, DHCP Agent
oscompute1 | 10.0.100.147 | 192.168.182.152 | nova-compute
oscompute2 | 10.0.100.148 | 192.168.182.153 | nova-compute
oskeystone  | 10.0.100.149 | 192.168.182.154 | qpid/rabbitmq, keystone, mysql
osswift0 | 10.0.100.139 | 192.168.182.144 | swift0
osswift1 | 10.0.100.140 | 192.168.182.145 | swift1
osswift2 | 10.0.100.141 | 192.168.182.146 | swift2
osceph0 | 10.0.100.142 | 192.168.182.147 | ceph0
osceph1 | 10.0.100.143 | 192.168.182.148 | ceph1
osceph2 | 10.0.100.144 | 192.168.182.149 | ceph2


![参考部署架构](http://docs.openstack.org/icehouse/install-guide/install/yum/content/figures/1/figures/installguide_arch-neutron.png)


## 基础环境配置
### Network staff
#### 配置远程登录，使用~/.ssh/config来简化远程登录操作（免密码验证）：

```bash
$ ssh-keygen -t rsa -P '' -f ~/.ssh/cloudlab150.key
$ ssh-copy-id -i ~/.ssh/cloudlab150.key.pub root@192.168.182.150 # 这里需要输入密码一次
```
编辑`~/.ssh/config`:

```
Host oscontroller
		HostName 192.168.182.150
		User root
		PreferredAuthentications publickey
		IdentityFile ~/.ssh/cloudlab150.key
```
```bash
$ ssh oscontroller
```
通过脚本自动生成Key并添加到~/.ssh/config：

```bash
$ for i in {144..154}; do ssh-keygen -t rsa -P '' -f ~/.ssh/cloudlab$i.key; ssh-copy-id -i ~/.ssh/cloudlab$i.key.pub root@192.168.182.$i ;done;
$ for i in {144..154}; do echo -e "Host cloudlab$i\n  HostName 192.168.182.$i\n  User root\n  PreferredAuthentications publickey\n  IdentityFile ~/.ssh/cloudlab$i.key" >> ~/.ssh/config;done;
```
编辑`~/.ssh/config`: 根据部署规划可以添加Host别名，方便记忆

```
...
Host cloudlab154 oskeystone
		HostName 192.168.182.154
...
```
### 配置主机HostName及/etc/hosts

```bash
$ vim /etc/sysconfig/network
$ hostname oskeystone # 一台台配置oscontroller, osnetwork...
```
编辑`/etc/hosts`: 可以统一配置然后通过scp等工具分发到所有节点

```
192.168.182.144 osswift0
192.168.182.145 osswift1
192.168.182.146 osswift2
192.168.182.147 osceph0
192.168.182.148 osceph1
192.168.182.149 osceph2
192.168.182.150 oscontroller
192.168.182.151 osnetwork
192.168.182.152 oscompute1
192.168.182.153 oscompute2
192.168.182.154 oskeystone
```

### 配置时区，启用NTP服务
在oskeystone上部署NTP服务器，其他节点从oskeystone上同步时间：

```bash
$ yum install ntp # 每个节点都需要ntp client
$ chkconfig ntpd on # oskeystone自动启动ntpd
```
编辑`/etc/ntpd.conf`: 
服务器端，即oskeystone:

```
...
server	202.112.10.60 # s1a.time.edu.cn
server	127.127.1.0     # local clock
fudge	127.127.1.0 stratum 10
restrict 10.0.100.0 mask 255.255.255.0 nomodify
...
```
客户端：

```
server 10.0.100.149
restrict 10.0.100.149 nomodify notrap noquery
```

启动ntpd服务：

```bash
$ iptables -F # 服务器端需要开放UDP 123端口，这里简单测试直接清空
$ ntpdate 10.0.100.149 # 手动同步一次
$ service ntpd start
$ ntpq -p # 检查时钟源
$ ntpstat # 检查时间同步状态
```
调整时区：

```bash
$ rm -f /etc/localtime
$ ln -s /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
```

### 密码准备
提前准备好安装过程中用到的密码：

```bash
for i in KEYSTONE GLANCE NOVA CINDER NEUTRON HEAT CEILOMETER TROVE; do echo $i"_DBPASS" `openssl rand -hex 10`; echo $i"_PASS" `openssl rand -hex 10`; done
for i in MYSQL_PASS QPID_PASS DEMO_PASS ADMIN_PASS DASH_DBPASS; do echo $i `openssl rand -hex 10`;done;
```
记录好生成的输出：

```
KEYSTONE_DBPASS 2e37d19cb04e42cf83ae
KEYSTONE_PASS 86be667d6555105f9eb6 #不需要
GLANCE_DBPASS 9fb6d0fddec4a6dcfa10
GLANCE_PASS 7cb6a024c60dd85c0b71
NOVA_DBPASS e5a43fc52ce0aaa1c7b4
NOVA_PASS fd53a063286a4f22ab7b
CINDER_DBPASS 791ce55fa6888c065bf3
CINDER_PASS 89edec5c2869dce44c61
NEUTRON_DBPASS fd535f9e2441a28a252d
NEUTRON_PASS 033d164bcc70e1244be7
HEAT_DBPASS 51eb27f53983633f3337
HEAT_PASS 3817bfface1b24918d4b
CEILOMETER_DBPASS 072aac393486f9b29235
CEILOMETER_PASS eb8ed86ef2178168c458
TROVE_DBPASS 4bf36df17323ff60ac43
TROVE_PASS 1342204826b34e479cb5
MYSQL_PASS a904019ba8cc0b14bef2
QPID_PASS 232a62982a3bcdc86cba
DEMO_PASS c3f7871eaa28ca146d09
ADMIN_PASS b517d18b5663e8e9c2ae
DASH_DBPASS 32ec0f799695e5ff6fe0
```

### 数据库安装配置
选择在oskeystone上安装mysql作为公共的数据库服务器：

```bash
$ yum install mysql mysql-server MySQL-python
$ yum install MySQL-python # 需要访问mysql的客户端需要安装python驱动，主要是oscontroller
```
编辑`/etc/my.cnf`：

```
[mysqld]
...
bind-address = 10.0.100.149
default-storage-engine = innodb
innodb_file_per_table
collation-server = utf8_general_ci
init-connect = 'SET NAMES utf8'
character-set-server = utf8
...
```
启动mysql数据库：

```
$ service mysqld start
$ chkconfig mysqld on
$ mysql_install_db
$ mysql_secure_installation
```

### 消息服务安装配置
在oskeystone安装Qpid消息服务器：

```bash
$ yum install qpid-cpp-server
```
暂时不做验证，编辑`/etc/qpidd.conf`:

```
auth=no
```
启动qpid服务端：

```bash
$ service qpidd start
$ chkconfig qpidd on
```


### yum第三方源添加

```bash
$ yum install -y yum-plugin-priorities
$ yum install -y http://repos.fedorapeople.org/repos/openstack/openstack-icehouse/rdo-release-icehouse-3.noarch.rpm
$ yum install -y http://dl.fedoraproject.org/pub/epel/6/x86_64/epel-release-6-8.noarch.rpm
```
验证第三方源：

```bash
$ yum install openstack-utils # 基于crudini，对于多个值的option，需要手动修改配置
$ yum install openstack-selinux
```
更新组件：

```bash
$ yum upgrade
$ reboot  # 如果更新了kernel，需要重启
```


## ID Service: Keystone
![keystone原理](http://docs.openstack.org/icehouse/install-guide/install/yum/content/figures/2/figures/SCH_5002_V00_NUAC-Keystone.png)
最关键的几个概念中，**tenant**最关键，它是一个资源容器，里面包含了用户、服务列表、组织、账户等资源，是用来聚集和隔离资源的单元。从第2、3步中可以看到，用户是先确定了tenant再获取对应的服务列表。而Keystone在整个过程中，对每一个步骤都进行了鉴权操作。

### 在oskeystone上安装keystone服务：

```bash
$ yum install openstack-keystone python-keystoneclient
$ openstack-config --set /etc/keystone/keystone.conf \
   database connection mysql://keystone:2e37d19cb04e42cf83ae@10.0.100.149/keystone
$ mysql -u root -p
mysql> CREATE DATABASE keystone;
mysql> GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'localhost' \
  IDENTIFIED BY 'KEYSTONE_DBPASS';
mysql> GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%' \
  IDENTIFIED BY 'KEYSTONE_DBPASS';
mysql> exit
$ chown -R keystone:keystone /var/log/keystone
$ chown -R keystone:keystone /etc/keystone/
$ su -s /bin/sh -c "keystone-manage db_sync" keystone
$ ADMIN_TOKEN=$(openssl rand -hex 10) # keystone和其他service之间的token
$ echo $ADMIN_TOKEN
$ openstack-config --set /etc/keystone/keystone.conf DEFAULT admin_token $ADMIN_TOKEN
$ keystone-manage pki_setup --keystone-user keystone --keystone-group keystone # 使用 PKI token，效率会低，但是安全。
$ chown -R keystone:keystone /etc/keystone/ssl
$ chmod -R o-rwx /etc/keystone/ssl
$ service openstack-keystone start
$ chkconfig openstack-keystone on
```
把过期的token从数据库删除，在log中记录，防止mysql性能恶化：

```bash
$ (crontab -l -u keystone 2>&1 | grep -q token_flush) || \
echo '@hourly /usr/bin/keystone-manage token_flush >/var/log/keystone/keystone-tokenflush.log 2>&1' >> /var/spool/cron/keystone
```
### 定义User, tenant, role
用ADMIN_TOKEN创建管理员和普通DEMO用户账号：

```bash
$ export OS_SERVICE_TOKEN=b412e43d938a38122d84 # 之前创建的ADMIN_TOKEN
$ export OS_SERVICE_ENDPOINT=http://oskeystone:35357/v2.0
$ keystone user-create --name=admin --pass=b517d18b5663e8e9c2ae --email=admin@tecstack.org # 密码开始时已批量生成
$ keystone role-create --name=admin
$ keystone tenant-create --name=admin --description="Admin Tenant"
$ keystone user-role-add --user=admin --tenant=admin --role=admin
$ keystone user-role-add --user=admin --role=_member_ --tenant=admin
$ keystone user-create --name=demo --pass=c3f7871eaa28ca146d09 --email=demo@tecstack.org
$ keystone tenant-create --name=demo --description="Demo Tenant"
$ keystone user-role-add --user=demo --role=_member_ --tenant=demo
```
创建service之间互访的tenant：

```bash
$ keystone tenant-create --name=service --description="Service Tenant"
```
在keystone上注册服务：

```bash
$ keystone service-create --name=keystone --type=identity \
  --description="OpenStack Identity"
$ keystone endpoint-create \
  --service-id=$(keystone service-list | awk '/ identity / {print $2}') \
  --publicurl=http://192.168.182.154:5000/v2.0 \
  --internalurl=http://10.0.100.149:5000/v2.0 \
  --adminurl=http://10.0.100.149:35357/v2.0
```
验证keystone服务是否注册成功：

```bash
$ unset OS_SERVICE_TOKEN OS_SERVICE_ENDPOINT
$ keystone --os-username=admin --os-password=b517d18b5663e8e9c2ae \
  --os-auth-url=http://10.0.100.149:35357/v2.0 token-get
$ keystone --os-username=admin --os-password=b517d18b5663e8e9c2ae \
  --os-tenant-name=admin --os-auth-url=http://10.0.100.149:35357/v2.0 \
  token-get	# 带tenant权限
```
新增一个环境变量设置文件`~/adminrc`简化客户端命令执行参数：

```bash
export OS_USERNAME=admin
export OS_PASSWORD=b517d18b5663e8e9c2ae
export OS_TENANT_NAME=admin
export OS_AUTH_URL=http://10.0.100.149:35357/v2.0
export PS1='[\u@\h \W(keystone_admin)]$ ' # 识别是否处于管理员状态
```
验证管理员权限：

```
$ source ~/adminrc
$ keystone user-list
```
同样也建立一个Demo普通用户的`~/demorc`：

```bash
export OS_USERNAME=demo
export OS_PASSWORD=c3f7871eaa28ca146d09
export OS_TENANT_NAME=demo
export OS_AUTH_URL=http://10.0.100.149:35357/v2.0
export PS1='[\u@\h \W(keystone_demo)]$ '
```

## Openstack Service Clients

Openstack的服务端都提供RESTful API，客户端都是通过curl方式进行交互，且都是基于python2.x实现。
cinder nova trove keystone glance neutron swift heat ceilometer
python-swiftclient
可以通过yum安装，也可以通过pip安装，也可以通过pyenv、virtualenv等创建隔离环境后安装以辅助开发多个版本，具体可以参考[Python开发环境搭建][os2]。这里直接安装到系统版本的python下：

```bash
$ yum groupinstall -y "Development Tools" # 有些依赖需要gcc编译
$ yum install -y python-devel
$ alias easy_install='easy_install -i http://pypi.douban.com/simple' # 用douban源加速
$ easy_install pip
$ for i in cinder nova trove keystone glance neutron swift heat ceilometer; do pip install  "python-"$i"client";done;
```
对于源比较慢的情况，可以考虑在一台机器上装完所有client包，然后建立pip源给其他机器使用。

```bash
$ yum install -y nginx
$ pip install pip2pi
$ mkdir -p /opt/pip/packages
$ 
```

## Images Service: Glance

## Compute Service: Nova

## Network Service: Neutron with ML2/VxLAN

## Block Service: Cinder with Ceph

## Object Service: Swift

## Dashboard

## Ceilometer

## Heat

## Trove


参考：
1. [Openstack docs][os1]
2. [Python开发环境搭建][os2]

[os1]:http://docs.openstack.org/icehouse/ "Openstack docs"
[os2]:http://promisejohn.github.io/2015/04/15/PythonDevEnvSetting/ "Python开发环境搭建"