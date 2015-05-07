---
title: "Openstack安装部署"
date: 2015-05-07 15:12:56
tags: [openstack,nova,neutron,cinder,glance,ceilometer,swift]
categories: [Tech]
---




## 参考架构及部署规划
![参考架构](http://docs.openstack.org/icehouse/install-guide/install/yum/content/figures/1/a/common/figures/openstack_havana_conceptual_arch.png)

* 操作系统：CentOS6.5
* 第三方yum源：epel, rdo

节点部署角色：

节点名 | internal ip | public ip | Role
----- | ------------| ----------| ----
oscontroller | 10.0.100.145 | 192.168.182.150 | nova, glance, cinder, image, neutron, dashboard, heat
osnetwork | 10.0.100.146 | 192.168.182.151 | ML2, OVS, L2 Agent, L3 Agent, DHCP Agent
oscompute1 | 10.0.100.147 | 192.168.182.152 | nova-compute
oscompute2 | 10.0.100.148 | 192.168.182.153 | nova-compute
oskeystone  | 10.0.100.149 | 192.168.182.154 | qpid/rabbitmq, keystone, mysql， memcached
osmeter  | 10.0.100.150 | 192.168.182.155 | ceilometer, mongodb
osswift0 | 10.0.100.139 | 192.168.182.144 | swift0, swift-proxy-server
osswift1 | 10.0.100.140 | 192.168.182.145 | swift1
osswift2 | 10.0.100.141 | 192.168.182.146 | swift2
osceph0 | 10.0.100.142 | 192.168.182.147 | ceph0
osceph1 | 10.0.100.143 | 192.168.182.148 | ceph1 # 暂时不用
osceph2 | 10.0.100.144 | 192.168.182.149 | ceph2 # 暂时不用


![参考部署架构](http://docs.openstack.org/icehouse/install-guide/install/yum/content/figures/1/figures/installguide_arch-neutron.png)

<!-- more -->

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
$ tzselect
$ TZ='Asia/Shanghai'; export TZ
```

### 密码准备
提前准备好安装过程中用到的密码：

```bash
for i in KEYSTONE GLANCE NOVA CINDER NEUTRON HEAT CEILOMETER; do echo $i"_DBPASS" `openssl rand -hex 10`; echo $i"_PASS" `openssl rand -hex 10`; done
for i in MYSQL_PASS QPID_PASS DEMO_PASS ADMIN_PASS DASH_DBPASS METADATA_SECRET SWIFT_PASS; do echo $i `openssl rand -hex 10`;done;
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
MYSQL_PASS a904019ba8cc0b14bef2
QPID_PASS 232a62982a3bcdc86cba
DEMO_PASS c3f7871eaa28ca146d09
ADMIN_PASS b517d18b5663e8e9c2ae
DASH_DBPASS 32ec0f799695e5ff6fe0
METADATA_SECRET 69e2f4db01fe4100bd32
SWIFT_PASS d8e42c6a28bf6eb08b76
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
可以通过yum安装，也可以通过pip安装，也可以通过pyenv、virtualenv等创建隔离环境后安装以辅助开发多个版本，具体可以参考[Python开发环境搭建][os2]。这里直接安装到系统版本的python下：

```bash
$ yum groupinstall -y "Development Tools" # 有些依赖需要gcc编译
$ yum install -y python-devel
$ alias easy_install='easy_install -i http://pypi.douban.com/simple' # 用douban源加速
$ easy_install pip
```
修改pip源，`~/.pip/pip.conf`:

```
[global]
index-url = http://pypi.douban.com/simple
```
批量安装Clients：

```bash
$ for i in cinder nova keystone glance neutron swift heat ceilometer; do pip install  "python-"$i"client";done;
```

## Images Service: Glance
默认采用本地文件系统存储镜像：`/var/lib/glance/images/`
glance-api提供API，glance-registry负责image的metadata存取查询。metadata保存在mysql，image文件本身可以存在文件系统或对象存储等。

### 安装Glance Service
在oscontroller上部署Glance Service：

```bash
$ yum install -y openstack-glance python-glanceclient
$ openstack-config --set /etc/glance/glance-api.conf database \
  connection mysql://glance:9fb6d0fddec4a6dcfa10@10.0.100.149/glance
$ openstack-config --set /etc/glance/glance-registry.conf database \
  connection mysql://glance:9fb6d0fddec4a6dcfa10@10.0.100.149/glance
```
在oskeystone上增加glance数据库：

```
$ mysql -u root -pa904019ba8cc0b14bef2
mysql> CREATE DATABASE glance;
mysql> GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'localhost' \
IDENTIFIED BY '9fb6d0fddec4a6dcfa10';
mysql> GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'%' \
IDENTIFIED BY '9fb6d0fddec4a6dcfa10';
```
在oscontroller上同步数据库：

```bash
$ chown -R glance:glance /etc/glance
$ chown -R glance:glance /var/log/glance
$ chown -R glance:glance /var/lib/glance
$ chown -R glance:glance /var/run/glance
$ su -s /bin/sh -c "glance-manage db_sync" glance # 使用glance账户执行命令
```
Ops，出现Warning和Error：

```
...PowmInsecureWarning: Not using mpz_powm_sec.  You should rebuild using libgmp >= 5 to avoid timing attack vulnerability.
  _warn("Not using mpz_powm_sec.  You should rebuild using libgmp >= 5 to avoid timing attack vulnerability.", PowmInsecureWarning)...
```
```  
$ tailf /var/log/glance/api.log
... glance AttributeError: 'NoneType' object has no attribute 'replace' ...
```

Warning可以在[这个网址][os3]查到解决方案：

```bash
$ wget https://gmplib.org/download/gmp/gmp-6.0.0a.tar.bz2
$ tar xvf gmp-6.0.0a.tar.bz2
$ cd gmp-6.0.0
$ ./configure
$ make
$ make check
$ make install
$ wget https://ftp.dlitz.net/pub/dlitz/crypto/pycrypto/pycrypto-2.6.1.tar.gz # 从源码安装，而不是从pip，否则会有ImportError: .../_AES.so: undefined symbol: rpl_malloc错误
$ tar xvf pycrypto-2.6.1.tar.gz
$ cd pycrypto-2.6.1
$ export ac_cv_func_malloc_0_nonnull=yes
$ ./configure
$ python setup.py build
$ python setup.py install
```
出现的错误很奇怪，网上也没什么案例，只能一步步看日志trace：

```
File "/usr/lib/python2.6/site-packages/glance/openstack/common/db/sqlalchemy/migration.py", line 236, in db_version
2015-04-22 02:43:28.196 27939 TRACE glance     meta.reflect(bind=engine)
2015-04-22 02:43:28.196 27939 TRACE glance   File "/usr/lib64/python2.6/site-packages/sqlalchemy/schema.py", line 2776, in reflect
2015-04-22 02:43:28.196 27939 TRACE glance     connection=conn))
2015-04-22 02:43:28.196 27939 TRACE glance   File "/usr/lib64/python2.6/site-packages/sqlalchemy/engine/base.py", line 1677, in table_names
2015-04-22 02:43:28.196 27939 TRACE glance     return self.dialect.get_table_names(conn, schema)
```
可以关注的是3处关键词：**bind=engine**, **connection=conn**, **get_table_names(conn, schema)**，很大程度上表明数据库连接建立失败，而且是在获取schema的时候，重新检查一下`/etc/glance/glance-api.conf`，发现mysql连接最后少加了schema，即误写成了`mysql://glance:9fb6d0fddec4a6dcfa10@10.0.100.149/`，少了`glance`，改正后再执行db_sync成功。


### 创建Glance服务权限：

创建Glance Service与Keystone验证的用户：

```bash
$ source ~/adminrc  # 加载admin环境变量
$ keystone user-create --name=glance --pass=7cb6a024c60dd85c0b71 \
   --email=glance@tecstack.org
$ keystone user-role-add --user=glance --tenant=service --role=admin
```

配置Glance Service通过Keystone进行鉴权：

```bash
$ openstack-config --set /etc/glance/glance-api.conf keystone_authtoken \
  auth_uri http://10.0.100.149:5000
$ openstack-config --set /etc/glance/glance-api.conf keystone_authtoken \
  auth_host 10.0.100.149
$ openstack-config --set /etc/glance/glance-api.conf keystone_authtoken \
  auth_port 35357
$ openstack-config --set /etc/glance/glance-api.conf keystone_authtoken \
  auth_protocol http
$ openstack-config --set /etc/glance/glance-api.conf keystone_authtoken \
  admin_tenant_name service
$ openstack-config --set /etc/glance/glance-api.conf keystone_authtoken \
  admin_user glance
$ openstack-config --set /etc/glance/glance-api.conf keystone_authtoken \
  admin_password 7cb6a024c60dd85c0b71
$ openstack-config --set /etc/glance/glance-api.conf paste_deploy \
  flavor keystone
$ openstack-config --set /etc/glance/glance-registry.conf keystone_authtoken \
  auth_uri http://10.0.100.149:5000
$ openstack-config --set /etc/glance/glance-registry.conf keystone_authtoken \
  auth_host 10.0.100.149
$ openstack-config --set /etc/glance/glance-registry.conf keystone_authtoken \
  auth_port 35357
$ openstack-config --set /etc/glance/glance-registry.conf keystone_authtoken \
  auth_protocol http
$ openstack-config --set /etc/glance/glance-registry.conf keystone_authtoken \
  admin_tenant_name service
$ openstack-config --set /etc/glance/glance-registry.conf keystone_authtoken \
  admin_user glance
$ openstack-config --set /etc/glance/glance-registry.conf keystone_authtoken \
  admin_password 7cb6a024c60dd85c0b71
$ openstack-config --set /etc/glance/glance-registry.conf paste_deploy \
  flavor keystone
```
注册Glance Service服务：

```bash
$ keystone service-create --name=glance --type=image \
  --description="OpenStack Image Service"
$ keystone endpoint-create \
  --service-id=$(keystone service-list | awk '/ image / {print $2}') \
  --publicurl=http://192.168.182.150:9292 \
  --internalurl=http://10.0.100.145:9292 \
  --adminurl=http://10.0.100.145:9292
```
启动Glance服务进程：

```bash
$ service openstack-glance-api start
$ service openstack-glance-registry start
$ chkconfig openstack-glance-api on
$ chkconfig openstack-glance-registry on
```

### 验证Glance Service
用cirros验证镜像服务：

```bash
$ wget http://cdn.download.cirros-cloud.net/0.3.2/cirros-0.3.2-x86_64-disk.img
$ file cirros-0.3.2-x86_64-disk.img
cirros-0.3.2-x86_64-disk.img: Qemu Image, Format: Qcow , Version: 2
$ source ~/adminrc
$ glance image-create --name "cirros-0.3.2-x86_64" --disk-format qcow2 \
  --container-format bare --is-public True --progress < cirros-0.3.2-x86_64-disk.img
$ glance image-list
```
如果不想下载镜像，可以直接通过`--copy-from`参数直接导入：

```bash
$ glance image-create --name="cirros-0.3.2-x86_64" --disk-format=qcow2 \
  --container-format=bare --is-public=true \
  --copy-from http://cdn.download.cirros-cloud.net/0.3.2/cirros-0.3.2-x86_64-disk.img
```

## Compute Service: Nova
Compute是IaaS的核心，`nova-api`,`nova-api-metadata`提供API服务；`nova-compute`调用hypervisor API启停虚拟机，`nova-scheduler`从请求队列调度分配主机，`nova-conductor`把`nova-compute`和数据库解耦，可以横向扩展，不建议跟`nova-compute`一起部署；`nova-network`, `nova-dhcpbridge`，网络部分功能逐步合并到Neutron Service；`nova-consoleauth`，负责控制台鉴权，`nova-novncproxy`和`nova-xvpnvncproxy`分别对应浏览器和java客户端，`nova-cert`管理x509证书。`nova-objectstore`和`euca2ools`是针对EC2场景的工具，实现Image Service和S3的互通；`nova`,`nova-manage`是本地客户端。

### Nova安装
在oscontroller上安装nova管理端：

```bash
$ yum install -y openstack-nova-api openstack-nova-cert openstack-nova-conductor openstack-nova-console openstack-nova-novncproxy openstack-nova-scheduler python-novaclient
$ openstack-config --set /etc/nova/nova.conf \
  database connection mysql://nova:e5a43fc52ce0aaa1c7b4@10.0.100.149/nova
$ openstack-config --set /etc/nova/nova.conf \
  DEFAULT rpc_backend qpid
$ openstack-config --set /etc/nova/nova.conf DEFAULT qpid_hostname 10.0.100.149
$ openstack-config --set /etc/nova/nova.conf DEFAULT my_ip 10.0.100.145
$ openstack-config --set /etc/nova/nova.conf DEFAULT vncserver_listen 10.0.100.145
$ openstack-config --set /etc/nova/nova.conf DEFAULT vncserver_proxyclient_address 10.0.100.145
```
在oskeystone上创建nova的数据库：

```bash
$ mysql -u root -pa904019ba8cc0b14bef2
mysql> CREATE DATABASE nova;
mysql> GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'%' \
IDENTIFIED BY 'e5a43fc52ce0aaa1c7b4';
```
在oscontroller上同步nova数据库：

```bash
$ chown -R nova:nova /etc/nova/
$ chown -R nova:nova /var/log/nova/
$ chown -R nova:nova /var/run/nova/
$ chown -R nova:nova /var/lib/nova/
$ su -s /bin/sh -c "nova-manage db sync" nova
```
创建Nova Service在Keystone上的账号权限，配置其使用keystone做鉴权：

```bash
$ keystone user-create --name=nova --pass=fd53a063286a4f22ab7b --email=nova@tecstack.org
$ keystone user-role-add --user=nova --tenant=service --role=admin
$ openstack-config --set /etc/nova/nova.conf DEFAULT auth_strategy keystone
$ openstack-config --set /etc/nova/nova.conf keystone_authtoken auth_uri http://10.0.100.149:5000
$ openstack-config --set /etc/nova/nova.conf keystone_authtoken auth_host 10.0.100.149
$ openstack-config --set /etc/nova/nova.conf keystone_authtoken auth_protocol http
$ openstack-config --set /etc/nova/nova.conf keystone_authtoken auth_port 35357
$ openstack-config --set /etc/nova/nova.conf keystone_authtoken admin_user nova
$ openstack-config --set /etc/nova/nova.conf keystone_authtoken admin_tenant_name service
$ openstack-config --set /etc/nova/nova.conf keystone_authtoken admin_password fd53a063286a4f22ab7b
```
在oskeystone上注册Nova Service：

```bash
$ source ~/adminrc
$ keystone service-create --name=nova --type=compute \
  --description="OpenStack Compute"
$ keystone endpoint-create \
  --service-id=$(keystone service-list | awk '/ compute / {print $2}') \
  --publicurl=http://192.168.182.150:8774/v2/%\(tenant_id\)s \
  --internalurl=http://10.0.100.145:8774/v2/%\(tenant_id\)s \
  --adminurl=http://10.0.100.145:8774/v2/%\(tenant_id\)s
```
启动Nova服务程序：

```bash
$ for svc in openstack-nova-{api,cert,consoleauth,scheduler,conductor,novncproxy}; do service $svc start; chkconfig $svc on; done;
$ for svc in openstack-nova-{api,cert,consoleauth,scheduler,conductor,novncproxy}; do service $svc status; chkconfig | grep $svc; done;  # 检查进程状态及自动启动设置状态
```
检查Nova Service部署

```bash
$ source ~/adminrc
$ nova image-list
```

### 部署计算节点：
在oscompute1和oscompute2上安装：

```bash
$ yum install -y openstack-nova-compute
$ egrep -c '(vmx|svm)' /proc/cpuinfo # 确定CPU支持虚拟化，如果不支持硬件虚拟化，需要配置nova.conf中的libvirt virt_type为qemu
# openstack-config --set /etc/nova/nova.conf libvirt virt_type qemu
```
编辑`/etc/nova/nova.conf`：

```bash
$ openstack-config --set /etc/nova/nova.conf database connection mysql://nova:e5a43fc52ce0aaa1c7b4@10.0.100.149/nova
$ openstack-config --set /etc/nova/nova.conf DEFAULT auth_strategy keystone
$ openstack-config --set /etc/nova/nova.conf keystone_authtoken auth_uri http://10.0.100.149:5000
$ openstack-config --set /etc/nova/nova.conf keystone_authtoken auth_host 10.0.100.149
$ openstack-config --set /etc/nova/nova.conf keystone_authtoken auth_protocol http
$ openstack-config --set /etc/nova/nova.conf keystone_authtoken auth_port 35357
$ openstack-config --set /etc/nova/nova.conf keystone_authtoken admin_user nova
$ openstack-config --set /etc/nova/nova.conf keystone_authtoken admin_tenant_name service
$ openstack-config --set /etc/nova/nova.conf keystone_authtoken admin_password fd53a063286a4f22ab7b
$ openstack-config --set /etc/nova/nova.conf DEFAULT rpc_backend qpid
$ openstack-config --set /etc/nova/nova.conf DEFAULT qpid_hostname 10.0.100.149
```
在计算节点启用console：

```bash
$ openstack-config --set /etc/nova/nova.conf DEFAULT my_ip 192.168.182.152/153
$ openstack-config --set /etc/nova/nova.conf DEFAULT vnc_enabled True
$ openstack-config --set /etc/nova/nova.conf DEFAULT vncserver_listen 0.0.0.0
$ openstack-config --set /etc/nova/nova.conf DEFAULT vncserver_proxyclient_address 192.168.182.152/153
$ openstack-config --set /etc/nova/nova.conf \
  DEFAULT novncproxy_base_url http://192.168.182.150:6080/vnc_auto.html
```
配置Glance：

```bash
$ openstack-config --set /etc/nova/nova.conf DEFAULT glance_host 10.0.100.145
```
启动进程服务：

```bash
$ service libvirtd start
$ service messagebus start
$ service openstack-nova-compute start
$ chkconfig libvirtd on
$ chkconfig messagebus on
$ chkconfig openstack-nova-compute on
```


## Network Service: Neutron with ML2
网络部分关键概念包括：networks, subnets, routes，至少有1个“外部”网络。网络功能由plugin提供，如core plugin, security group plugin, FW, LB, ...

### 部署Neutron管理端
#### 在oskeystone上准备Neutron数据库：

```bash
$ mysql -u root -p
mysql> CREATE DATABASE neutron;
mysql> GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'%' \
IDENTIFIED BY 'fd535f9e2441a28a252d';
```
创建Neutron的keystone账号，并注册Neutron服务：

```bash
$ source ~/adminrc
$ keystone user-create --name neutron --pass 033d164bcc70e1244be7 --email neutron@tecstack.org
$ keystone user-role-add --user neutron --tenant service --role admin
$ keystone service-create --name neutron --type network --description "OpenStack Networking"
$ keystone endpoint-create \
  --service-id $(keystone service-list | awk '/ network / {print $2}') \
  --publicurl http://192.168.182.150:9696 \
  --adminurl http://10.0.100.145:9696 \
  --internalurl http://10.0.100.145:9696
```
#### 在oscontroller上部署配置Neutron服务端：

```bash
$ yum install -y openstack-neutron openstack-neutron-ml2 python-neutronclient
$ openstack-config --set /etc/neutron/neutron.conf database connection \
  mysql://neutron:fd535f9e2441a28a252d@10.0.100.149/neutron
$ openstack-config --set /etc/neutron/neutron.conf DEFAULT \
  auth_strategy keystone
$ openstack-config --set /etc/neutron/neutron.conf keystone_authtoken \
  auth_uri http://10.0.100.149:5000
$ openstack-config --set /etc/neutron/neutron.conf keystone_authtoken \
  auth_host 10.0.100.149
$ openstack-config --set /etc/neutron/neutron.conf keystone_authtoken \
  auth_protocol http
$ openstack-config --set /etc/neutron/neutron.conf keystone_authtoken \
  auth_port 35357
$ openstack-config --set /etc/neutron/neutron.conf keystone_authtoken \
  admin_tenant_name service
$ openstack-config --set /etc/neutron/neutron.conf keystone_authtoken \
  admin_user neutron
$ openstack-config --set /etc/neutron/neutron.conf keystone_authtoken \
  admin_password 033d164bcc70e1244be7
$ openstack-config --set /etc/neutron/neutron.conf DEFAULT \
  core_plugin ml2
$ openstack-config --set /etc/neutron/neutron.conf DEFAULT \
  service_plugins router
```
配置Neutron使用MQ：

```bash
$ openstack-config --set /etc/neutron/neutron.conf DEFAULT \
  rpc_backend neutron.openstack.common.rpc.impl_qpid
$ openstack-config --set /etc/neutron/neutron.conf DEFAULT \
  qpid_hostname 10.0.100.149
```
配置Neutron与计算节点的通知：

```bash
$ openstack-config --set /etc/neutron/neutron.conf DEFAULT \
  notify_nova_on_port_status_changes True
$ openstack-config --set /etc/neutron/neutron.conf DEFAULT \
  notify_nova_on_port_data_changes True
$ openstack-config --set /etc/neutron/neutron.conf DEFAULT \
  nova_url http://10.0.100.145:8774/v2
$ openstack-config --set /etc/neutron/neutron.conf DEFAULT \
  nova_admin_username nova
$ openstack-config --set /etc/neutron/neutron.conf DEFAULT \
  nova_admin_tenant_id $(keystone tenant-list | awk '/ service / { print $2 }')
$ openstack-config --set /etc/neutron/neutron.conf DEFAULT \
  nova_admin_password fd53a063286a4f22ab7b
$ openstack-config --set /etc/neutron/neutron.conf DEFAULT \
  nova_admin_auth_url http://10.0.100.149:35357/v2.0
```
创建neutron ml2 plugn配置链接：

```bash
$ ln -s plugins/ml2/ml2_conf.ini /etc/neutron/plugin.ini
```

配置ML2插件：ML2使用OVS处理网络流量。在oscontroller上配置：

```bash
$ openstack-config --set /etc/neutron/plugins/ml2/ml2_conf.ini ml2 \
  type_drivers gre
$ openstack-config --set /etc/neutron/plugins/ml2/ml2_conf.ini ml2 \
  tenant_network_types gre
$ openstack-config --set /etc/neutron/plugins/ml2/ml2_conf.ini ml2 \
  mechanism_drivers openvswitch
$ openstack-config --set /etc/neutron/plugins/ml2/ml2_conf.ini ml2_type_gre \
  tunnel_id_ranges 1:1000
$ openstack-config --set /etc/neutron/plugins/ml2/ml2_conf.ini securitygroup \
  firewall_driver neutron.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver
$ openstack-config --set /etc/neutron/plugins/ml2/ml2_conf.ini securitygroup \
  enable_security_group True
```

#### 在oscontroller上：配置Nova管理端使用Neutron管理网络：

```bash
$ openstack-config --set /etc/nova/nova.conf DEFAULT \
  network_api_class nova.network.neutronv2.api.API
$ openstack-config --set /etc/nova/nova.conf DEFAULT \
  neutron_url http://10.0.100.145:9696
$ openstack-config --set /etc/nova/nova.conf DEFAULT \
  neutron_auth_strategy keystone
$ openstack-config --set /etc/nova/nova.conf DEFAULT \
  neutron_admin_tenant_name service
$ openstack-config --set /etc/nova/nova.conf DEFAULT \
  neutron_admin_username neutron
$ openstack-config --set /etc/nova/nova.conf DEFAULT \
  neutron_admin_password 033d164bcc70e1244be7
$ openstack-config --set /etc/nova/nova.conf DEFAULT \
  neutron_admin_auth_url http://10.0.100.149:35357/v2.0
$ openstack-config --set /etc/nova/nova.conf DEFAULT \
  linuxnet_interface_driver nova.network.linux_net.LinuxOVSInterfaceDriver
$ openstack-config --set /etc/nova/nova.conf DEFAULT \
  firewall_driver nova.virt.firewall.NoopFirewallDriver # 禁用nova-compute内置的FW功能。
$ openstack-config --set /etc/nova/nova.conf DEFAULT \
  security_group_api neutron
```

在oscontroller上重启Nova服务：

```bash
$ service openstack-nova-api restart
$ service openstack-nova-scheduler restart
$ service openstack-nova-conductor restart
```
#### 在oscontroller上启动Neutron服务：

```bash
$ service neutron-server start
$ chkconfig neutron-server on
```
正常情况下，neutron-server在启动的时候会自动初始化数据库，如果出错，可以在oscontroller上手动建立数据库：

```bash
$ openstack-config --set /etc/neutron/neutron.conf DEFAULT \
  core_plugin neutron.plugins.ml2.plugin.Ml2Plugin
$ openstack-config --set /etc/neutron/neutron.conf DEFAULT \
  service_plugins neutron.services.l3_router.l3_router_plugin.L3RouterPlugin
$ su -s /bin/sh -c "neutron-db-manage --config-file /etc/neutron/neutron.conf \
  --config-file /etc/neutron/plugin.ini upgrade head" neutron
```

### 在osnetwork上配置Linux转发及Neutron Agent
#### 启用IP转发
编辑`/etc/sysctl.conf`：

```
net.ipv4.ip_forward=1
net.ipv4.conf.all.rp_filter=0
net.ipv4.conf.default.rp_filter=0
net.bridge.bridge-nf-call-arptables=1
net.bridge.bridge-nf-call-iptables=1
net.bridge.bridge-nf-call-ip6tables=1
```
使生效：

```bash
$ sysctl -p
```
#### 部署配置Neutron及ML2, OVS, L3：

```bash
$ yum install -y openstack-neutron openstack-neutron-ml2 \
  openstack-neutron-openvswitch
$ openstack-config --set /etc/neutron/neutron.conf DEFAULT \
  auth_strategy keystone
$ openstack-config --set /etc/neutron/neutron.conf keystone_authtoken \
  auth_uri http://10.0.100.149:5000
$ openstack-config --set /etc/neutron/neutron.conf keystone_authtoken \
  auth_host 10.0.100.149
$ openstack-config --set /etc/neutron/neutron.conf keystone_authtoken \
  auth_protocol http
$ openstack-config --set /etc/neutron/neutron.conf keystone_authtoken \
  auth_port 35357
$ openstack-config --set /etc/neutron/neutron.conf keystone_authtoken \
  admin_tenant_name service
$ openstack-config --set /etc/neutron/neutron.conf keystone_authtoken \
  admin_user neutron
$ openstack-config --set /etc/neutron/neutron.conf keystone_authtoken \
  admin_password 033d164bcc70e1244be7
$ openstack-config --set /etc/neutron/neutron.conf DEFAULT \
  rpc_backend neutron.openstack.common.rpc.impl_qpid
$ openstack-config --set /etc/neutron/neutron.conf DEFAULT \
  qpid_hostname 10.0.100.149
$ openstack-config --set /etc/neutron/neutron.conf DEFAULT \
  core_plugin ml2
$ openstack-config --set /etc/neutron/neutron.conf DEFAULT \
  service_plugins router
$ openstack-config --set /etc/neutron/l3_agent.ini DEFAULT \
  interface_driver neutron.agent.linux.interface.OVSInterfaceDriver
$ openstack-config --set /etc/neutron/l3_agent.ini DEFAULT \
  use_namespaces True
```
官方文档对于网络包大小解释的比较清楚，直接引用如下：

>
Tunneling protocols such as generic routing encapsulation (GRE) include additional packet headers that increase overhead and decrease space available for the payload or user data. Without knowledge of the virtual network infrastructure, instances attempt to send packets using the default Ethernet maximum transmission unit **(MTU) of 1500 bytes**. Internet protocol (IP) networks contain the path MTU discovery (**PMTUD**) mechanism to detect end-to-end MTU and adjust packet size accordingly. However, some operating systems and networks block or otherwise **lack support for PMTUD** causing performance degradation or connectivity failure.
>
Ideally, you can prevent these problems by enabling **jumbo frames on the physical network** that contains your tenant virtual networks. Jumbo frames support MTUs up to approximately **9000 bytes** which negates the impact of GRE overhead on virtual networks. However, many network devices lack support for jumbo frames and OpenStack administrators often lack control of network infrastructure. Given the latter complications, you can also prevent MTU problems by **reducing the instance MTU to account for GRE overhead**. Determining the proper MTU value often takes experimentation, but **1454 bytes** works in most environments. You can configure the DHCP server that assigns IP addresses to your instances to also adjust the MTU.
Some cloud images such as CirrOS ignore the DHCP MTU option.

大致意思是默认的MTU为1500字节，当加入了GRE等tunnel包头后会超过1500字节，IP协议理论上通过PMTUD可以自动调节传输路径上的MTU大小，但有些OS和网络设备因缺乏对PMTUD的支持会拦截此类包导致通断或性能问题。Jumbo frames支持最大9000字节的MTU，但是需要硬件网络设备的支持。
通常情况下，可以从虚拟机端设计MTU的大小，一般1454字节（减去了GRE的包头大小）在大部分环境下可行，MTU的设置可以通过DHCP服务器配置，但有些OS如CirrOS不支持DHCP配置MTU。

#### 配置osnetwork上的dhcp组件：

```bash
$ openstack-config --set /etc/neutron/dhcp_agent.ini DEFAULT \
  interface_driver neutron.agent.linux.interface.OVSInterfaceDriver
$ openstack-config --set /etc/neutron/dhcp_agent.ini DEFAULT \
  dhcp_driver neutron.agent.linux.dhcp.Dnsmasq
$ openstack-config --set /etc/neutron/dhcp_agent.ini DEFAULT \
  use_namespaces True
$ openstack-config --set /etc/neutron/dhcp_agent.ini DEFAULT \
  dnsmasq_config_file /etc/neutron/dnsmasq-neutron.conf # DHCP MTU
```
新建编辑`/etc/neutron/dnsmasq-neutron.conf`：

```
dhcp-option-force=26,1454
```
停止所有dnsmasq进程：

```bash
killall dnsmasq
```

#### 配置metadata agent组件：

```bash
$ openstack-config --set /etc/neutron/metadata_agent.ini DEFAULT \
  auth_url http://10.0.100.149:5000/v2.0
$ openstack-config --set /etc/neutron/metadata_agent.ini DEFAULT \
  auth_region regionOne
$ openstack-config --set /etc/neutron/metadata_agent.ini DEFAULT \
  admin_tenant_name service
$ openstack-config --set /etc/neutron/metadata_agent.ini DEFAULT \
  admin_user neutron
$ openstack-config --set /etc/neutron/metadata_agent.ini DEFAULT \
  admin_password 033d164bcc70e1244be7
$ openstack-config --set /etc/neutron/metadata_agent.ini DEFAULT \
  nova_metadata_ip 10.0.100.145
$ openstack-config --set /etc/neutron/metadata_agent.ini DEFAULT \
  metadata_proxy_shared_secret 69e2f4db01fe4100bd32
```

#### 配置oscontroller上的nova：

```bash
$ openstack-config --set /etc/nova/nova.conf DEFAULT \
  service_neutron_metadata_proxy true
$ openstack-config --set /etc/nova/nova.conf DEFAULT \
  neutron_metadata_proxy_shared_secret 69e2f4db01fe4100bd32
$ service openstack-nova-api restart
```

#### 配置osnetwork上的ML2, OVS插件：

```bash
$ openstack-config --set /etc/neutron/plugins/ml2/ml2_conf.ini ml2 \
  type_drivers gre
$ openstack-config --set /etc/neutron/plugins/ml2/ml2_conf.ini ml2 \
  tenant_network_types gre
$ openstack-config --set /etc/neutron/plugins/ml2/ml2_conf.ini ml2 \
  mechanism_drivers openvswitch
$ openstack-config --set /etc/neutron/plugins/ml2/ml2_conf.ini ml2_type_gre \
  tunnel_id_ranges 1:1000
$ openstack-config --set /etc/neutron/plugins/ml2/ml2_conf.ini ovs \
  local_ip 10.0.100.146 # osnetwork的内部ip
$ openstack-config --set /etc/neutron/plugins/ml2/ml2_conf.ini ovs \
  tunnel_type gre
$ openstack-config --set /etc/neutron/plugins/ml2/ml2_conf.ini ovs \
  enable_tunneling True
$ openstack-config --set /etc/neutron/plugins/ml2/ml2_conf.ini securitygroup \
  firewall_driver neutron.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver
$ openstack-config --set /etc/neutron/plugins/ml2/ml2_conf.ini securitygroup \
  enable_security_group True
$ ln -s plugins/ml2/ml2_conf.ini /etc/neutron/plugin.ini
```

修复打包的bug，让neutron-openvswitch-agent使用`plugin.ini`配置文件：

```bash
$ cp /etc/init.d/neutron-openvswitch-agent /etc/init.d/neutron-openvswitch-agent.orig
$ sed -i 's,plugins/openvswitch/ovs_neutron_plugin.ini,plugin.ini,g' /etc/init.d/neutron-openvswitch-agent
```

#### 启动OVS，添加br-int, br-ex为内部和外部bridge：

```bash
$ service openvswitch start
$ chkconfig openvswitch on
$ ovs-vsctl add-br br-int
$ ovs-vsctl add-br br-ex
$ ovs-vsctl add-port br-ex eth1
```
官网提示内部虚拟机和外部网络之间的吞吐可以通过关闭GRO提升，临时关闭方法：`ethtool -K INTERFACE_NAME gro off`


Ops. 发现添加port后网络连接断开了。Google下，Linux bridge kernel module一直要求一旦某个interface加入到了在某个bridge下时，不能再单独配置IP被访问，如果要访问host，则可以把这个IP（或其他可访问的IP）分配给所在的bridge。可以参考[这个链接][os4]。对应的解决方案就是把原来的public ip从eth1删除，配置到br-exs上：

```bash
$ ip addr del 192.168.182.151/25 dev eth1
$ ip addr add 192.168.182.151/25 dev br-ex
$ ip route del default via 192.168.182.254 dev eth1  # 改变默认路由
$ ip route add default via 192.168.182.254 dev br-ex
```

这样就能再次访问osnetwork节点，看[RDO的文档][os5]里也有提及，建议把br-ex写到配置文件并删除eth1配置文件里的IP信息后重启网络：

新增`/etc/sysconfig/network-scripts/ifcfg-br-ex`:

```
DEVICE=br-ex
DEVICETYPE=ovs
TYPE=OVSBridge
BOOTPROTO=static
IPADDR=192.168.182.151
NETMASK=255.255.255.128 # your netmask
GATEWAY=192.168.182.254 # your gateway
#DNS1=192.168.122.1      # your nameserver
ONBOOT=yes
```
修改`/etc/sysconfig/network-scripts/ifcfg-eth1`:

```
DEVICE=eth1
HWADDR=00:50:56:b8:dd:e0 # your hwaddr
TYPE=OVSPort
DEVICETYPE=ovs
OVS_BRIDGE=br-ex
ONBOOT=yes

#NM_CONTROLLED=yes
#BOOTPROTO=none
#IPADDR=192.168.182.151
#NETMASK=255.255.255.128
#GATEWAY=192.168.182.254
#IPV6INIT=no
#USERCTL=no
```
修改完之后，执行：`service network restart`，系统会自动把默认路由指向br-ex。


#### 启动Neutron Agent服务：

```bash
$ for svc in neutron-{openvswitch,l3,dhcp,metadata}-agent; do \
	service $svc start; chkconfig $svc on;done;
$ for svc in neutron-{openvswitch,l3,dhcp,metadata}-agent; do \
	service $svc status; chkconfig | grep  $svc;done;
```

### 在oscompute1和oscompute2上配置Linux转发及Neutron Agent

#### 启用Linux转发
编辑`/etc/sysctl.conf`：

```bash
net.ipv4.conf.all.rp_filter=0
net.ipv4.conf.default.rp_filter=0
net.bridge.bridge-nf-call-arptables=1
net.bridge.bridge-nf-call-iptables=1
net.bridge.bridge-nf-call-ip6tables=1
```
使生效：

```bash
$ sysctl -p
```

#### 安装部署Neutron ML2和OVS

```bash
$ yum install -y openstack-neutron-ml2 openstack-neutron-openvswitch
$ openstack-config --set /etc/neutron/neutron.conf DEFAULT \
  auth_strategy keystone
$ openstack-config --set /etc/neutron/neutron.conf keystone_authtoken \
  auth_uri http://10.0.100.149:5000
$ openstack-config --set /etc/neutron/neutron.conf keystone_authtoken \
  auth_host 10.0.100.149
$ openstack-config --set /etc/neutron/neutron.conf keystone_authtoken \
  auth_protocol http
$ openstack-config --set /etc/neutron/neutron.conf keystone_authtoken \
  auth_port 35357
$ openstack-config --set /etc/neutron/neutron.conf keystone_authtoken \
  admin_tenant_name service
$ openstack-config --set /etc/neutron/neutron.conf keystone_authtoken \
  admin_user neutron
$ openstack-config --set /etc/neutron/neutron.conf keystone_authtoken \
  admin_password 033d164bcc70e1244be7
$ openstack-config --set /etc/neutron/neutron.conf DEFAULT \
  rpc_backend neutron.openstack.common.rpc.impl_qpid
$ openstack-config --set /etc/neutron/neutron.conf DEFAULT \
  qpid_hostname 10.0.100.149
$ openstack-config --set /etc/neutron/neutron.conf DEFAULT \
  core_plugin ml2
$ openstack-config --set /etc/neutron/neutron.conf DEFAULT \
  service_plugins router
```
#### 配置ML2

```bash
$ openstack-config --set /etc/neutron/plugins/ml2/ml2_conf.ini ml2 \
  type_drivers gre
$ openstack-config --set /etc/neutron/plugins/ml2/ml2_conf.ini ml2 \
  tenant_network_types gre
$ openstack-config --set /etc/neutron/plugins/ml2/ml2_conf.ini ml2 \
  mechanism_drivers openvswitch
$ openstack-config --set /etc/neutron/plugins/ml2/ml2_conf.ini ml2_type_gre \
  tunnel_id_ranges 1:1000
$ openstack-config --set /etc/neutron/plugins/ml2/ml2_conf.ini ovs \
  local_ip 10.0.100.147/148
$ openstack-config --set /etc/neutron/plugins/ml2/ml2_conf.ini ovs \
  tunnel_type gre
$ openstack-config --set /etc/neutron/plugins/ml2/ml2_conf.ini ovs \
  enable_tunneling True
$ openstack-config --set /etc/neutron/plugins/ml2/ml2_conf.ini securitygroup \
  firewall_driver neutron.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver
$ openstack-config --set /etc/neutron/plugins/ml2/ml2_conf.ini securitygroup \
  enable_security_group True
$ ln -s plugins/ml2/ml2_conf.ini /etc/neutron/plugin.ini
```

#### 配置OVS

```bash
$ service openvswitch start
$ chkconfig openvswitch on
$ ovs-vsctl add-br br-int
```

#### 配置Nova-Compute

```bash
$ openstack-config --set /etc/nova/nova.conf DEFAULT \
  network_api_class nova.network.neutronv2.api.API
$ openstack-config --set /etc/nova/nova.conf DEFAULT \
  neutron_url http://10.0.100.145:9696
$ openstack-config --set /etc/nova/nova.conf DEFAULT \
  neutron_auth_strategy keystone
$ openstack-config --set /etc/nova/nova.conf DEFAULT \
  neutron_admin_tenant_name service
$ openstack-config --set /etc/nova/nova.conf DEFAULT \
  neutron_admin_username neutron
$ openstack-config --set /etc/nova/nova.conf DEFAULT \
  neutron_admin_password 033d164bcc70e1244be7
$ openstack-config --set /etc/nova/nova.conf DEFAULT \
  neutron_admin_auth_url http://10.0.100.149:35357/v2.0
$ openstack-config --set /etc/nova/nova.conf DEFAULT \
  linuxnet_interface_driver nova.network.linux_net.LinuxOVSInterfaceDriver
$ openstack-config --set /etc/nova/nova.conf DEFAULT \
  firewall_driver nova.virt.firewall.NoopFirewallDriver
$ openstack-config --set /etc/nova/nova.conf DEFAULT \
  security_group_api neutron
```

#### 修改`neutron-openvswitch-agent`使用plugin.ini配置：

```bash
$ cp /etc/init.d/neutron-openvswitch-agent /etc/init.d/neutron-openvswitch-agent.orig
$ sed -i 's,plugins/openvswitch/ovs_neutron_plugin.ini,plugin.ini,g' /etc/init.d/neutron-openvswitch-agent
```

#### 启动`nova-computer`,`neutron-openvswitch-agent`进程

```bash
$ service openstack-nova-compute restart
$ service neutron-openvswitch-agent start
$ chkconfig neutron-openvswitch-agent on
```

### 初始化虚拟机的网络
启动第一个虚拟机实例之前，必须先创建给虚拟机连接的虚拟网络，包括外部网络和租户网络。
在oscontroller上，通过admin账户创建外部网络，共享给所有租户；通过demo账户创建租户私有网络，再创建路由实现与外部网络连通。

```bash
$ source adminrc
$ neutron net-create ext-net --shared --router:external=True
$ neutron subnet-create ext-net --name ext-subnet \
  --allocation-pool start=192.168.182.200,end=192.168.182.220 \
  --disable-dhcp --gateway 192.168.182.254 192.168.182.128/25
$ source demorc
$ neutron net-create demo-net
$ neutron subnet-create demo-net --name demo-subnet \
  --gateway 192.168.1.1 192.168.1.0/24
$ neutron router-create demo-router
$ neutron router-interface-add demo-router demo-subnet
$ ip netns list # 能看到新增了一个`qrouter-4bd0dc16-ae05-4304-9483-41c0f9a67775`的namespace
$ ip netns exec qrouter-4bd0dc16-ae05-4304-9483-41c0f9a67775 ip addr # 能看到新增了`qr-af814b8f-c9`这个port。
$ ovs-vsctl list-ports br-int # 能看到增加了`qr-af814b8f-c9`这个port。
$ neutron router-gateway-set demo-router ext-net
$ ip netns exec qrouter-4bd0dc16-ae05-4304-9483-41c0f9a67775 ip addr # 能看到新增了`qg-269d911c-25`这个port，且它的ip就是ext-subnet里的floating IP。
$ ovs-vsctl list-ports br-ex # 能看到增加了`qg-269d911c-25`这个port。
```
检验连通性，在osnetwork上ping所创建的router（第一个floating IP地址）：

```bash
$ ping -c 4 192.168.182.200
$ ip netns exec qrouter-4bd0dc16-ae05-4304-9483-41c0f9a67775 ping 192.168.182.151  # 从router发起ping到外部的网络，首先可以ping通所在物理机osnetwork的端口
$ ip netns exec qrouter-4bd0dc16-ae05-4304-9483-41c0f9a67775 ping 192.168.182.100 # 可以ping通网关和其他子网下的Hosts
```
从这里可以理解，底层是通过Linux kernel的net namespace实现了网络隔离，即多租户能力。其实neutron给router添加interface和设定gateway的的时候，其实是在底层创建了一个net namespace并把租户子网的网关和从外部网络获取的floatingIP添加到了router中作为port，从而实现租户内私有网络和外部公共网络打通。

如果发现从router里可以和osnetwork互相ping通，但是无法和同一子网下的其他主机ping通，很大程度上是二层交换问题，可以通过在osnetwork和router上同时抓包，分析arp请求；如果发现arp无返回，则说明二层osnetwork之外的交换机配置有问题，比如在vmware虚拟化环境下安装openstack，则需要把vmware的虚拟交换机开启`混杂模式`。用到的调测命令如下，更多可以参考[网络问题排查方法][os6]：

```bash
$ ovs-appctl fdb/show br-ex # 查看br-ex的mac表，可以查看mac地址学习状态
$ ip netns exec qrouter-4bd0dc16-ae05-4304-9483-41c0f9a67775 tcpdump -nn arp  # 在router上抓arp包
$ ip netns exec qrouter-4bd0dc16-ae05-4304-9483-41c0f9a67775 ip route # 在router上执行路由命令
$ ovs-vsctl list Interface gre-0a006491 # 查看 ovs交换机上某个端口的详细状态
$ neutron agent-list # 查看neutron上网络组件状态
$ ovs-dpctl show #查看ovs端口配置
```

### 虚拟机网络流量通道
![ovs networking](http://docs.openstack.org/admin-guide-cloud/content/figures/14/a/a/common/figures/under-the-hood-scenario-1-ovs-compute.png)
在创建虚拟机之后（可以部署完dashboard之后做更方便），可以`ip addr`看到oscomputeX上新增了4个设备，分别以`tap, qbr, qvb, qvo`开头，osnetwork上也也新增了`qdhcp`开头的namespace。oscomputeX上新增的tap设备用于连接虚拟机端口，qbr是一个普通linux bridge，qvb和qvo是veth pair，连接了qbr和br-int。不把tap直接挂到br-int的理由就是目前Openstack security group的实现方式是基于iptables对tap设备的控制，而openvswitch不支持iptables控制挂在它上面的tap设备。
官方引文如下，详细可以参考[Networking Openvswitch on Openstack][os7]:

> Security groups: iptables and Linux bridges
Ideally, the TAP device vnet0 would be connected directly to the integration bridge, br-int. Unfortunately, this isn't possible because of how OpenStack security groups are currently implemented. OpenStack uses iptables rules on the TAP devices such as vnet0 to implement security groups, and Open vSwitch is not compatible with iptables rules that are applied directly on TAP devices that are connected to an Open vSwitch port.
Networking uses an extra Linux bridge and a veth pair as a workaround for this issue. Instead of connecting vnet0 to an Open vSwitch bridge, it is connected to a Linux bridge, qbrXXX. This bridge is connected to the integration bridge, br-int, through the (qvbXXX, qvoXXX) veth pair.

### 删除或变更GRE通道
当使用GRE tunnel时，所有network节点和compute节点之间是建立了full-mesh结构的点对点tunnel，当需要变动ip地址的时候（比如之前配置的时候把network节点上的local_ipx写错），需要手动把neutron数据库ip_addressb表中的错误ip删除，修改配置文件后重启network和计算节点傻姑上的`neutron-openvswitch-agent`服务。


## Dashboard
Horizon是一个基于python2.6的Django app。需要安装Keystone和Nova服务之后才能使用dashboard。
在oskeystone上部署dashboard：

```bash
$ yum install -y memcached python-memcached mod_wsgi openstack-dashboard
```
根据`/etc/sysconfig/memcached`内容修改`/etc/openstack-dashboard/local_settings`：

```
CACHES = {
	'default': {
	'BACKEND' : 'django.core.cache.backends.memcached.MemcachedCache',
	'LOCATION' : '127.0.0.1:11211'
	}
}
ALLOWED_HOSTS = ['localhost', '192.168.182.154']
OPENSTACK_HOST = "10.0.100.149"  # hostname of Identity Service
```
启动服务：

```bash
$ setsebool -P httpd_can_network_connect on
$ getsebool httpd_can_network_connect # check selinux status
$ service httpd start
$ service httpd start
$ chkconfig httpd on
$ chkconfig memcached on
```

访问`http://192.168.182.149/dashboard`，Ops...看apache日志`tailf /var/log/httpd/error_log`：

```
...
File "/usr/lib/python2.6/site-packages/keystoneclient/__init__.py", line 43, in <module>
__version__ = pbr.version.VersionInfo('python-keystoneclient').version_string()
...
File "/usr/lib/python2.6/site-packages/pbr/packaging.py", line 864, in get_version
raise Exception("Versioning for this project requires either an sdist"
Exception: Versioning for this project requires either an sdist tarball, or access to an upstream git repository. Are you sure that git is installed?
...
```
看起来跟`python-keystoneclient`的加载有关，于是关闭apache，用本地命令执行方式启动horizon的Django应用，如果要切换回Apache等启动方式，记得删除`/tmp`下自动生成的`SECRET_KEY`（在`/etc/openstack-dashboard/local_settings`中配置）：

```bash
$ cd /usr/share/openstack-dashboard
$ python manage.py runserver 0.0.0.0:80
```
发现错误信息一致，甚至不加参数执行`python manage.py`也会出现同样错误。Google到[一个同样的问题](https://github.com/rackspace/pyrax/issues/450)，更新`distribute`可解决，具体原因还没找到，初步估计跟打包方式及pbr有关：

```
$ pip install --upgrade distribute
```
另，如果出现`ERROR: [Errno 113] No route to host`，可以检查Nova服务端的Host上iptables是否设置正确。


## Block Service: Cinder
* `cinder-api`: API
* `cinder-volume`: 通过driver与块设备交互
* `cinder-scheduler`: 选择最优块来创建卷


### 在oscontroller上安装Cinder Service

```bash
$ yum install -y openstack-cinder
$ openstack-config --set /etc/cinder/cinder.conf \
  database connection mysql://cinder:791ce55fa6888c065bf3@10.0.100.149/cinder
```

在oskeystone上创建cinder的数据库：

```bash
$ mysql -u root -pa904019ba8cc0b14bef2
mysql> CREATE DATABASE cinder;
mysql> GRANT ALL PRIVILEGES ON cinder.* TO 'cinder'@'%' \
  IDENTIFIED BY '791ce55fa6888c065bf3';
```
同步数据库结构：

```bash
$ su -s /bin/sh -c "cinder-manage db sync" cinder
```
创建cinder的keystone账户，配置`/etc/cinder/cinder.conf`：

```bash
$ keystone user-create --name=cinder --pass=89edec5c2869dce44c61 --email=cinder@tecstack.org
$ keystone user-role-add --user=cinder --tenant=service --role=admin
$ openstack-config --set /etc/cinder/cinder.conf DEFAULT \
  auth_strategy keystone
$ openstack-config --set /etc/cinder/cinder.conf keystone_authtoken \
  auth_uri http://10.0.100.149:5000
$ openstack-config --set /etc/cinder/cinder.conf keystone_authtoken \
  auth_host 10.0.100.149
$ openstack-config --set /etc/cinder/cinder.conf keystone_authtoken \
  auth_protocol http
$ openstack-config --set /etc/cinder/cinder.conf keystone_authtoken \
  auth_port 35357
$ openstack-config --set /etc/cinder/cinder.conf keystone_authtoken \
  admin_user cinder
$ openstack-config --set /etc/cinder/cinder.conf keystone_authtoken \
  admin_tenant_name service
$ openstack-config --set /etc/cinder/cinder.conf keystone_authtoken \
  admin_password 89edec5c2869dce44c61
```
配置MQ，在Keystone上注册Cinder Service：

```bash
$ openstack-config --set /etc/cinder/cinder.conf \
  DEFAULT rpc_backend qpid
$ openstack-config --set /etc/cinder/cinder.conf \
  DEFAULT qpid_hostname 10.0.100.149
# API Version 1
$ keystone service-create --name=cinder --type=volume --description="OpenStack Block Storage"
$ keystone endpoint-create \
  --service-id=$(keystone service-list | awk '/ volume / {print $2}') \
  --publicurl=http://192.168.182.150:8776/v1/%\(tenant_id\)s \
  --internalurl=http://10.0.100.145:8776/v1/%\(tenant_id\)s \
  --adminurl=http://10.0.100.145:8776/v1/%\(tenant_id\)s
# API Version 2
$ keystone service-create --name=cinderv2 --type=volumev2 --description="OpenStack Block Storage v2"
$ keystone endpoint-create \
  --service-id=$(keystone service-list | awk '/ volumev2 / {print $2}') \
  --publicurl=http://192.168.182.150:8776/v2/%\(tenant_id\)s \
  --internalurl=http://10.0.100.145:8776/v2/%\(tenant_id\)s \
  --adminurl=http://10.0.100.145:8776/v2/%\(tenant_id\)s
```
启动服务：

```bash
$ service openstack-cinder-api start
$ service openstack-cinder-scheduler start
$ chkconfig openstack-cinder-api on
$ chkconfig openstack-cinder-scheduler on
```

### 在osceph0上部署一个临时存储节点
安装配置cinder：

```bash
$ yum install -y openstack-cinder scsi-target-utils
$ openstack-config --set /etc/cinder/cinder.conf DEFAULT \
  auth_strategy keystone
$ openstack-config --set /etc/cinder/cinder.conf keystone_authtoken \
  auth_uri http://10.0.100.149:5000
$ openstack-config --set /etc/cinder/cinder.conf keystone_authtoken \
  auth_host 10.0.100.149
$ openstack-config --set /etc/cinder/cinder.conf keystone_authtoken \
  auth_protocol http
$ openstack-config --set /etc/cinder/cinder.conf keystone_authtoken \
  auth_port 35357
$ openstack-config --set /etc/cinder/cinder.conf keystone_authtoken \
  admin_user cinder
$ openstack-config --set /etc/cinder/cinder.conf keystone_authtoken \
  admin_tenant_name service
$ openstack-config --set /etc/cinder/cinder.conf keystone_authtoken \
  admin_password 89edec5c2869dce44c61
$ openstack-config --set /etc/cinder/cinder.conf \
  DEFAULT rpc_backend qpid
$ openstack-config --set /etc/cinder/cinder.conf \
  DEFAULT qpid_hostname 10.0.100.149
$ openstack-config --set /etc/cinder/cinder.conf \
  database connection mysql://cinder:791ce55fa6888c065bf3@10.0.100.149/cinder
$ openstack-config --set /etc/cinder/cinder.conf \
  DEFAULT my_ip 10.0.100.142
$ openstack-config --set /etc/cinder/cinder.conf \
  DEFAULT glance_host 10.0.100.145  # cinder使用glance images创建可启动卷
$ openstack-config --set /etc/cinder/cinder.conf \
  DEFAULT iscsi_helper tgtadm  # 使用tgtadm iSCSI service
```
配置`/etc/tgt/targets.conf`，让`iSCSI target service`发现卷：

```bash
include /etc/cinder/volumes/*
```
模拟块设备:

```bash
$ dd if=/dev/zero of=/opt/volumes/cinderVolumes bs=1M count=10240
$ losetup -f cinderVolumes # 自动找到一个未使用的loop设备
$ losetup -a  # 查看映射状态
$ pvcreate /dev/loop0 # Use your own loop device
$ vgcreate cinder-volumes /dev/loop0 # Use your own loop device
```
编辑`/etc/lvm/lvm.conf`，让LVM不要扫描VM所用的卷：

```
devices {
...
filter = [ "a/loop0/", "r/.*/"] # pvdisplay只能看到loop0的设备
...
}
```

启动服务：

```bash
$ service openstack-cinder-volume start
$ service tgtd start
$ chkconfig openstack-cinder-volume on
$ chkconfig tgtd on
```
验证测试：

```bash
$ source demorc
$ cinder create --display-name myVolume 1
$ cinder list # 如果状态为available则正常
```
之后就可以在dashboard上把卷attach到instance上当块设备使用。

## 创建Instance云主机
基于dashboard的很简单，看GUI操作很明确，基于命令也可以。

```bash
$ source demorc
$ ssh-keygen -t rsa -P '' -f my.key
$ nova keypair-add --pub-key ./my.key.pub demo-key
$ nova keypair-list
$ nova flavor-list
$ nova image-list
$ nova net-list
$ nova secgroup-list
$ nova boot --flavor m1.tiny --image cirros-0.3.2-x86_64 --nic net-id=b7d34c60-5439-4ec8-9468-8e2406801b98 \
  --security-group default --key-name demo-key demo-instance1
$ nova list
$ nova get-vnc-console demo-instance1 novnc
```

## Object Service: Swift

* swift-proxy-server: API，操作metadata, container，提供file和container列表，可以配合memcached提高性能；
* swift-account-server: swift账号管理
* swift-container-server: 容器管理
* swift-object-server: 存储节点，要求支持`XATTRS`，推荐`XFS`
* 定时任务进程，用于清理、一致性检验等

### 创建swift账号，在keystone上注册服务：
192.168.182.128/25做为public网络，10.0.100.0/24做为数据内部同步网络。

```bash
$ keystone user-create --name=swift --pass=d8e42c6a28bf6eb08b76 \
  --email=swift@tecstack.org
$ keystone user-role-add --user=swift --tenant=service --role=admin
$ keystone service-create --name=swift --type=object-store \
  --description="OpenStack Object Storage"
$ keystone endpoint-create \
  --service-id=$(keystone service-list | awk '/ object-store / {print $2}') \
  --publicurl='http://192.168.182.144:8080/v1/AUTH_%(tenant_id)s' \
  --internalurl='http://10.0.100.139:8080/v1/AUTH_%(tenant_id)s' \
  --adminurl=http://10.0.100.139:8080
```
在**所有节点**上配置swift hash，用于确定swift ring的mapping，所有节点要一致且不可更改:

```bash
$ mkdir -p /etc/swift
$ openssl rand -base64 12 # 执行2次，生成2个随机字符串，用于swift_hash_path_*fix
$ cat > /etc/swift/swift.conf << EOF
[swift-hash]
# random unique string that can never change (DO NOT LOSE)
swift_hash_path_prefix = Fjt8GhQTi13Vr6Hc
swift_hash_path_suffix = 7uOcSBwQrK6pAtfx
EOF
```

### 部署存储节点
每个osswift节点创建3个模拟的块设备，每个设备分配5G容量，在所有osswift上操作：

```bash
$ mkdir -p /opt/swift_disk
$ dd if=/dev/zero of=/opt/swift_disk/swdisk0 bs=1M count=5120
$ dd if=/dev/zero of=/opt/swift_disk/swdisk1 bs=1M count=5120
$ dd if=/dev/zero of=/opt/swift_disk/swdisk2 bs=1M count=5120
$ losetup -f /opt/swift_disk/swdisk0
$ losetup -f /opt/swift_disk/swdisk1
$ losetup -f /opt/swift_disk/swdisk2
$ losetup -a # 查看映射状态
```
安装部署：

```bash
$ yum install -y openstack-swift-account openstack-swift-container \
  openstack-swift-object xfsprogs xinetd
$ for i in loop{0,1,2};do mkfs.xfs /dev/$i; echo "/dev/$i /srv/node/$i xfs noatime,nodiratime,nobarrier,logbufs=8 0 0" >> /etc/fstab; mkdir -p /srv/node/$i ; mount /srv/node/$i;done;
$ chown -R swift:swift /srv/node  # 注意每次重建xfs之后需要确认权限
```
创建`/etc/rsyncd.conf`:

```
uid = swift
gid = swift
log file = /var/log/rsyncd.log
pid file = /var/run/rsyncd.pid
address = 10.0.100.139/140/141 # 各个存储节点的数据同步网IP
 
[account]
max connections = 2
path = /srv/node/
read only = false
lock file = /var/lock/account.lock
 
[container]
max connections = 2
path = /srv/node/
read only = false
lock file = /var/lock/container.lock
 
[object]
max connections = 2
path = /srv/node/
read only = false
lock file = /var/lock/object.lock
```
配置`/etc/xinetd.d/rsync`:

```
disable = no
```
启动xinetd:

```bash
$ service xinetd start
$ mkdir -p /var/swift/recon
$ chown -R swift:swift /var/swift/recon
```

配置`/etc/swift/account-server.conf`,`/etc/swift/container-server.conf`,`/etc/swift/object-server.conf`：

```
...
bind_ip = 192.168.182.144/145/146 # 绑定到各自节点的public网络地址
...
```

###在osswift0上部署proxy server
使用oskeystone上的memcached：

```bash
$ yum install -y openstack-swift-proxy python-swiftclient python-keystone-auth-token
```
配置`/etc/swift/proxy-server.conf`:

```bash
...
[filter:authtoken]
paste.filter_factory = keystoneclient.middleware.auth_token:filter_factory
# Delaying the auth decision is required to support token-less
# usage for anonymous referrers ('.r:*').
delay_auth_decision = true
auth_protocol = http
auth_host = 10.0.100.149
auth_port = 35357
admin_tenant_name = service
admin_user = swift
admin_password = d8e42c6a28bf6eb08b76

[filter:cache]
use = egg:swift	#memcache
10.0.100.149:11211
...
```

配置`/etc/swift/object-expirer.conf`:

```
...
[filter:cache]
use = egg:swift#memcache
memcache_servers = 10.0.100.149:11211
...
```


配置swift的ring：

```bash
$ cd /etc/swift
$ swift-ring-builder account.builder create 18 3 1
$ swift-ring-builder container.builder create 18 3 1
$ swift-ring-builder object.builder create 18 3 1
$ for i in loop{0,1,2}; do swift-ring-builder account.builder add z1-192.168.182.144:6002R10.0.100.139:6005/$i 100; swift-ring-builder container.builder add z1-192.168.182.144:6001R10.0.100.139:6004/$i 100; swift-ring-builder object.builder add z1-192.168.182.144:6000R10.0.100.139:6003/$i 100; done; # 把osswift0的设备加入zone1
$ for i in loop{0,1,2}; do swift-ring-builder account.builder add z2-192.168.182.145:6002R10.0.100.140:6005/$i 100; swift-ring-builder container.builder add z2-192.168.182.145:6001R10.0.100.140:6004/$i 100; swift-ring-builder object.builder add z2-192.168.182.145:6000R10.0.100.140:6003/$i 100; done; # 把osswift1的设备加入zone2
$ for i in loop{0,1,2}; do swift-ring-builder account.builder add z3-192.168.182.146:6002R10.0.100.141:6005/$i 100; swift-ring-builder container.builder add z3-192.168.182.146:6001R10.0.100.141:6004/$i 100; swift-ring-builder object.builder add z3-192.168.182.146:6000R10.0.100.141:6003/$i 100; done; # 把osswift2的设备加入zone3
$ swift-ring-builder account.builder  # 注意检查上面的IP, 端口信息
$ swift-ring-builder container.builder
$ swift-ring-builder object.builder # 检查
$ swift-ring-builder account.builder rebalance
$ swift-ring-builder container.builder rebalance
$ swift-ring-builder object.builder rebalance # Relance the ring.
$ chown -R swift:swift /etc/swift
```
把`account.ring.gz`,`container.ring.gz`,`object.ring.gz`复制到proxy-server和Storage Server的`/etc/swift`下。

```bash
$ cd /etc/swift # 由于proxy server部署在osswift0，在osswift0上生成ring配置文件。
$ scp account.ring.gz container.ring.gz  object.ring.gz root@osswift1:/etc/swift/
$ scp account.ring.gz container.ring.gz  object.ring.gz root@osswift2:/etc/swift/
```

###启动服务：
启动proxy节点上的服务：

```bash
$ service openstack-swift-proxy start
$ chkconfig openstack-swift-proxy on
```
启动存储节点上的服务：

```
$ for service in \
  openstack-swift-object openstack-swift-object-replicator openstack-swift-object-updater openstack-swift-object-auditor \
  openstack-swift-container openstack-swift-container-replicator openstack-swift-container-updater openstack-swift-container-auditor \
  openstack-swift-account openstack-swift-account-replicator openstack-swift-account-reaper openstack-swift-account-auditor; do \
    service $service start; chkconfig $service on; done # 也可以用swift-init all start启动所有服务进程
```
检验效果：（确保osswift各个节点的iptables配置正确）

```bash
$ source adminrc
$ swift stat
$ cd; echo "Hello test" > test.txt; echo "Hello test2" > test2.txt
$ swift upload myfiles test.txt
$ swift upload myfiles test2.txt
$ swift download myfiles
```
swift默认的log比较奇怪，是在/var/log/messages里。

## Heat
流程编排服务，类似于vagrant，适合自动化系统规划部署：

* heat: client
* heat-api: REST API
* heat-api-cfn: AWS Query API
* heat-engine: 根据模板执行

在oskeystone上准备heat的数据库：

```bash
$ mysql -u root -pa904019ba8cc0b14bef2
mysql> CREATE DATABASE heat;
mysql> GRANT ALL PRIVILEGES ON heat.* TO 'heat'@'%' \
IDENTIFIED BY '51eb27f53983633f3337';
```

在oscontroller上安装heat：

```bash
$ yum install -y openstack-heat-api openstack-heat-engine \
  openstack-heat-api-cfn
$ openstack-config --set /etc/heat/heat.conf \
  database connection mysql://heat:51eb27f53983633f3337@10.0.100.149/heat
$ openstack-config --set /etc/heat/heat.conf DEFAULT qpid_hostname 10.0.100.149
$ openstack-config --set /etc/heat/heat.conf keystone_authtoken \
  auth_uri http://10.0.100.149:5000/v2.0
$ openstack-config --set /etc/heat/heat.conf keystone_authtoken \
  auth_host 10.0.100.149  # 官方
$ openstack-config --set /etc/heat/heat.conf keystone_authtoken \
  auth_port 35357
$ openstack-config --set /etc/heat/heat.conf keystone_authtoken \
  auth_protocol http
$ openstack-config --set /etc/heat/heat.conf keystone_authtoken \
  admin_tenant_name service
$ openstack-config --set /etc/heat/heat.conf keystone_authtoken \
  admin_user heat
$ openstack-config --set /etc/heat/heat.conf keystone_authtoken \
  admin_password 3817bfface1b24918d4b
$ openstack-config --set /etc/heat/heat.conf ec2authtoken \
  auth_uri http://10.0.100.149:5000/v2.0  # 可以暂时不配置。
$ su -s /bin/sh -c "heat-manage db_sync" heat
```

创建Heat Service的keystone账号，注册服务：

```bash
$ keystone user-create --name=heat --pass=3817bfface1b24918d4b \
  --email=heat@tecstack.org
$ keystone user-role-add --user=heat --tenant=service --role=admin
$ keystone service-create --name=heat --type=orchestration \
  --description="Orchestration"
$ keystone endpoint-create \
  --service-id=$(keystone service-list | awk '/ orchestration / {print $2}') \
  --publicurl=http://192.168.182.150:8004/v1/%\(tenant_id\)s \
  --internalurl=http://10.0.100.145:8004/v1/%\(tenant_id\)s \
  --adminurl=http://10.0.100.145:8004/v1/%\(tenant_id\)s
$ keystone service-create --name=heat-cfn --type=cloudformation \
  --description="Orchestration CloudFormation"
$ keystone endpoint-create \
  --service-id=$(keystone service-list | awk '/ cloudformation / {print $2}') \
  --publicurl=http://192.168.182.150:8000/v1 \
  --internalurl=http://10.0.100.145:8000/v1 \
  --adminurl=http://10.0.100.145:8000/v1
$ keystone role-create --name heat_stack_user # Orchestration创建用户的默认角色
$ openstack-config --set /etc/heat/heat.conf \
  DEFAULT heat_metadata_server_url http://10.0.100.145:8000
$ openstack-config --set /etc/heat/heat.conf DEFAULT \ 	heat_waitcondition_server_url http://10.0.100.145:8000/v1/waitcondition
```
启动服务：

```bash
$ service openstack-heat-api start
$ service openstack-heat-api-cfn start
$ service openstack-heat-engine start
$ chkconfig openstack-heat-api on
$ chkconfig openstack-heat-api-cfn on
$ chkconfig openstack-heat-engine on
```

验证效果：

```bash
$ source demorc
$ vim test-stack.yml
```

```
heat_template_version: 2013-05-23

description: Test Template

parameters:
  ImageID:
    type: string
    description: Image use to boot a server
  NetID:
    type: string
    description: Network ID for the server

resources:
  server1:
    type: OS::Nova::Server
    properties:
      name: "Test server"
      image: { get_param: ImageID }
      flavor: "m1.tiny"
      networks:
      - network: { get_param: NetID }

outputs:
  server1_private_ip:
    description: IP address of the server in the private network
    value: { get_attr: [ server1, first_address ] }
```

```bash
$ NET_ID=$(nova net-list | awk '/ demo-net / { print $2 }')
$ heat stack-create -f test-stack.yml \
  -P "ImageID=cirros-0.3.2-x86_64;NetID=$NET_ID" testStack
$ heat stack-list
```



## Ceilometer
监控CPU、网络等指标，通过REST API访问。
* ceilometer-agent-compute: 部署在每个计算节点，目前主要针对计算节点采集信息；
* ceilometer-agent-central: 非计算节点信息采集；
* ceilometer-collector: 采集信息汇聚。
* ceilometer-alarm-notifier: 告警设置。
* ceilometer-api: 接受查询请求。
* 后端存储，比如mongodb。

### 在osmeter部署Ceilmeter Service：


```bash
$ yum install -y openstack-ceilometer-api openstack-ceilometer-collector \
  openstack-ceilometer-notification openstack-ceilometer-central openstack-ceilometer-alarm python-ceilometerclient
```
配置MongoDB：

```bash
$ yum install -y mongodb-server mongodb
$ vim /etc/mongodb.conf # edit the bind_ip to osmeter internal ip:10.0.100.150
$ service mongod start
$ chkconfig mongod on
$ mongo --host 10.0.100.150 --eval '
db = db.getSiblingDB("ceilometer");
db.addUser({user: "ceilometer",
            pwd: "072aac393486f9b29235",
            roles: [ "readWrite", "dbAdmin" ]})'
```
配置Ceilmeter服务：

```bash
$ openstack-config --set /etc/ceilometer/ceilometer.conf \
  database connection mongodb://ceilometer:072aac393486f9b29235@10.0.100.150:27017/ceilometer
$ CEILOMETER_TOKEN=$(openssl rand -hex 10)
$ echo $CEILOMETER_TOKEN
$ openstack-config --set /etc/ceilometer/ceilometer.conf publisher metering_secret $CEILOMETER_TOKEN
$ openstack-config --set /etc/ceilometer/ceilometer.conf \
  DEFAULT rpc_backend ceilometer.openstack.common.rpc.impl_qpid
$ openstack-config --set /etc/ceilometer/ceilometer.conf DEFAULT qpid_hostname 10.0.100.149
$ openstack-config --set /etc/ceilometer/ceilometer.conf \
  DEFAULT auth_strategy keystone
$ openstack-config --set /etc/ceilometer/ceilometer.conf \
  keystone_authtoken auth_host 10.0.100.149
$ openstack-config --set /etc/ceilometer/ceilometer.conf \
  keystone_authtoken admin_user ceilometer
$ openstack-config --set /etc/ceilometer/ceilometer.conf \
  keystone_authtoken admin_tenant_name service
$ openstack-config --set /etc/ceilometer/ceilometer.conf \
  keystone_authtoken auth_protocol http
$ openstack-config --set /etc/ceilometer/ceilometer.conf \
  keystone_authtoken auth_uri http://10.0.100.149:5000
$ openstack-config --set /etc/ceilometer/ceilometer.conf \
  keystone_authtoken admin_password eb8ed86ef2178168c458
$ openstack-config --set /etc/ceilometer/ceilometer.conf \
  service_credentials os_auth_url http://10.0.100.149:5000/v2.0
$ openstack-config --set /etc/ceilometer/ceilometer.conf \
  service_credentials os_username ceilometer
$ openstack-config --set /etc/ceilometer/ceilometer.conf \
  service_credentials os_tenant_name service
$ openstack-config --set /etc/ceilometer/ceilometer.conf \
  service_credentials os_password eb8ed86ef2178168c458
```
注册Keystone服务：

```bash
$ keystone service-create --name=ceilometer --type=metering \
  --description="Telemetry"
$ keystone endpoint-create \
  --service-id=$(keystone service-list | awk '/ metering / {print $2}') \
  --publicurl=http://192.168.182.155:8777 \
  --internalurl=http://10.0.100.150:8777 \
  --adminurl=http://10.0.100.150:8777
$ keystone user-create --name=ceilometer --pass=eb8ed86ef2178168c458 --email=ceilometer@tecstack.org
$ keystone user-role-add --user=ceilometer --tenant=service --role=admin
```
启动服务：

```bash
$ for svc in openstack-ceilometer-{api,notification,central,collector,alarm-evaluator,alarm-notifier}; do service $svc start; chkconfig $svc on; done;
```

### 给计算节点部署Agent：
在oscompute1和oscompute2上安装：

```bash
$ yum install -y openstack-ceilometer-compute python-ceilometerclient python-pecan
$ 
```
编辑`/etc/nova/nova.conf`:

```bash
$ openstack-config --set /etc/nova/nova.conf DEFAULT \
  instance_usage_audit True
$ openstack-config --set /etc/nova/nova.conf DEFAULT \
  instance_usage_audit_period hour
$ openstack-config --set /etc/nova/nova.conf DEFAULT \
  notify_on_state_change vm_and_task_state
```
多值的参数，直接修改文件如下：

```
[DEFAULT]
...
notification_driver = nova.openstack.common.notifier.rpc_notifier
notification_driver = ceilometer.compute.nova_notifier
... # qpid 和qpid_hostname之前已经配置过，这里采用一样
```
编辑`/etc/ceilometer/ceilometer.conf`:

```bash
$ openstack-config --set /etc/ceilometer/ceilometer.conf publisher \
  metering_secret 711e1a83f278a83de1f8 # 在ceilometer上用openssl生成的
$ openstack-config --set /etc/ceilometer/ceilometer.conf DEFAULT rpc_backend ceilometer.openstack.common.rpc.impl_qpid
# openstack-config --set /etc/ceilometer/ceilometer.conf DEFAULT qpid_hostname 10.0.100.149
$ openstack-config --set /etc/ceilometer/ceilometer.conf \
  keystone_authtoken auth_host 10.0.100.149
$ openstack-config --set /etc/ceilometer/ceilometer.conf \
  keystone_authtoken admin_user ceilometer
$ openstack-config --set /etc/ceilometer/ceilometer.conf \
  keystone_authtoken admin_tenant_name service
$ openstack-config --set /etc/ceilometer/ceilometer.conf \
  keystone_authtoken auth_protocol http
$ openstack-config --set /etc/ceilometer/ceilometer.conf \
  keystone_authtoken admin_password eb8ed86ef2178168c458
$ openstack-config --set /etc/ceilometer/ceilometer.conf \
  service_credentials os_username ceilometer
$ openstack-config --set /etc/ceilometer/ceilometer.conf \
  service_credentials os_tenant_name service
$ openstack-config --set /etc/ceilometer/ceilometer.conf \
  service_credentials os_password eb8ed86ef2178168c458
$ openstack-config --set /etc/ceilometer/ceilometer.conf \
  service_credentials os_auth_url http://10.0.100.149:5000/v2.0
```

重启服务：

```bash
$ service openstack-nova-compute restart
$ service openstack-ceilometer-compute start
$ chkconfig openstack-ceilometer-compute on
```


### 给Glance、Cinder、Swift部署Agent：

在oscontroller上编辑`/etc/glance/glance-api.conf`，并重启服务:

```bash
$ openstack-config --set /etc/glance/glance-api.conf DEFAULT notification_driver messaging
$ openstack-config --set /etc/glance/glance-api.conf DEFAULT rpc_backend qpid
$ openstack-config --set /etc/glance/glance-api.conf DEFAULT qpid_hostname 10.0.100.149
$ service openstack-glance-api restart
$ service openstack-glance-registry restart
```

在oscontroller和osceph0上编辑`/etc/cinder/cinder.conf`，并重启服务：

```bash
$ openstack-config --set /etc/cinder/cinder.conf DEFAULT control_exchange cinder
$ openstack-config --set /etc/cinder/cinder.conf DEFAULT notification_driver cinder.openstack.common.notifier.rpc_notifier
$ service openstack-cinder-api restart # 在oscontoller上
$ service openstack-cinder-scheduler restart # 在oscontroller上
$ service openstack-cinder-volume restart # 在osceph0上
```

在osswift0上安装：

```bash
$ yum install -y python-ceilometer
```

另，ceilometer需要`ResellerAdmin`角色来获取数据：

```bash
$ keystone role-create --name=ResellerAdmin
$ keystone user-role-add --tenant service --user ceilometer \
      --role $(keystone role-list | awk '/ ResellerAdmin / {print $2}')
```

在osswift0上修改`/etc/swift/proxy-server.conf`:

```
...
[filter:ceilometer]
use = egg:ceilometer#swift
...
[pipeline:main]
pipeline = healthcheck cache authtoken keystoneauth ceilometer proxy-server
...
```
重启`openstack-swift-proxy `服务:

```bash
$ service openstack-swift-proxy restart
```

### 检验ceilometer效果

```bash
$ ceilometer meter-list
$ glance image-download "cirros-0.3.2-x86_64" > cirros.img # 下载一个镜像
$ ceilometer meter-list
$ ceilometer statistics -m image.download -p 60
```

如果遇到采集指标不齐问题，可以检查MQ上的消息队列，可以用`qpid-tools`工具（其他MQ对应也有其他工具）：

```bash
$ yum install -y qpid-tools
$ qpid-tool 10.0.100.149
qpid > list
qpid > list queue/exchange
qpid > show ID_xxx
```

## 总结

Openstack主要的组件部署不复杂，文档目前也已经比较成熟，只不过涉及到linux和python dev相关内容较多，细节容易出错，多动动手基本也可以很快熟悉起来。但如果要深入内部，还得从log、运行原理等切入，直至跟进组件社区开发状态和进展，非一日之功。


参考：
1. [Openstack docs][os1]
2. [Python开发环境搭建][os2]
3. [openstack安装错误方案][os3]
4. [Problem configuring openvswitch br-ex][os4]
5. [Neutron_with_existing_external_network][os5]
6. [网络问题排查方法][os6]
7. [Networking Openvswitch on Openstack][os7]

[os1]:http://docs.openstack.org/icehouse/ "Openstack docs"
[os2]:http://promisejohn.github.io/2015/04/15/PythonDevEnvSetting/ "Python开发环境搭建"
[os3]:http://techglimpse.com/openstack-installation-errors-solutions/ "openstack安装错误方案"
[os4]:http://www.gossamer-threads.com/lists/openstack/operators/34627 "Problem configuring openvswitch br-ex"
[os5]:https://www.rdoproject.org/Neutron_with_existing_external_network "Neutron_with_existing_external_network"
[os6]:http://docs.openstack.org/openstack-ops/content/network_troubleshooting.html "Network troubleshooting"
[os7]:http://docs.openstack.org/admin-guide-cloud/content/under_the_hood_openvswitch.html "Networking Openvswitch on Openstack"