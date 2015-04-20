---
title: "Hello Vagrant"
date: 2015-04-17 10:22:06
tags: [vagrant, 虚拟化]
categories: [Tech]
---

### Vagrant部署

```bash
$ cd ~/dev
$ wget https://dl.bintray.com/mitchellh/vagrant/vagrant_1.7.2_x86_64.rpm
$ rpm -ivh vagrant_1.7.2_x86_64.rpm
```

### 添加vagrant boxes

```bash
$ vagrant box add hashicorp/precise32 # 支持virtualbox Hypervisor
$ vagrant box add centos64 http://citozin.com/centos64.box ＃ 支持 libvirt KVM Hypervisor
```

### 编辑Vagrantfile
默认使用的是virtualbox作为Hypervisor：

```ruby
Vagrant.configure("2") do |config|
  config.vm.box = "hashicorp/precise32"
end
```

### 运行虚拟机

```bash
$ vagrant up
```


### Vagrant使用KVM Hypervisor
安装vagrant-libvirt plugin：

```bash
$ yum install libxslt-devel libxml2-devel libvirt-devel
$ vagrant plugin install vagrant-libvirt --plugin-source https://ruby.taobao.org/
```
配置Vagrantfile：

```ruby
Vagrant.configure("2") do |config|
  config.vm.define :node1 do |node1|
    node1.vm.box = "centos64"
  end
end
```
启动虚拟机：

```bash
$ vagrant up --provider=libvirt
$ export VAGRANT_DEFAULT_PROVIDER=libvirt # Another way
```

### 小结

Vagrant的box不是对所有hypervisor都通用，比如官方的`hashicorp/precise32`就不支持KVM，如果环境只有KVM，那么可能需要自己制作对应的box，可以参考[Vagrant-Libvirt Plugin][vagrant1]，借助工具`create_box.sh`从qcow2 image制作。

此外，需要注意的是，KVM和VirtualBox不能同时启动虚拟机，否则会报类似如下错误：

先启动virtualbox虚拟机再启动KVM虚拟机：

```
There was an error talking to Libvirt. The error message is shown
below:

Call to virDomainCreateWithFlags failed: internal error Process exited while reading console log output: char device redirected to /dev/pts/2
kvm_create_vm: Device or resource busy
failed to initialize KVM: Operation not permitted
No accelerator found!
```
先启动KVM虚拟机再启动virtualbox虚拟机：

```
The guest machine entered an invalid state while waiting for it
to boot. Valid states are 'starting, running'. The machine is in the
'poweroff' state. Please verify everything is configured
properly and try again.

If the provider you're using has a GUI that comes with it,
it is often helpful to open that and watch the machine, since the
GUI often has more helpful error messages than Vagrant can retrieve.
For example, if you're using VirtualBox, run `vagrant up` while the
VirtualBox GUI is open.
```

参考：

1. [Vagrant Official Site][vagrant2]
2. [Vagrant-Libvirt Plugin][vagrant1]

[vagrant1]: https://github.com/pradels/vagrant-libvirt/ "Vagrant-Libvirt"
[vagrant2]: https://docs.vagrantup.com/v2/ "Vagrant Official Site"
[vagrant3]: http://promisejohn.github.io/2015/04/17/RubyDevEnvSetting/ "Ruby开发环境配置"
