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

### 编辑Vagrantfile
默认使用的是virtualbox作为Hypervisor：

```ruby
Vagrant.configure("2") do |config|
  config.vm.define :test_vm do |test_vm|
    test_vm.vm.box = "ubuntu"
  end
end
```


### Vagrant使用KVM Hypervisor
安装vagrant-libvirt plugin：

```bash
$ yum install libxslt-devel libxml2-devel libvirt-devel
$ vagrant plugin install vagrant-libvirt --plugin-source https://ruby.taobao.org/
$ vagrant up --provider=libvirt # OR use way bellow
$ export VAGRANT_DEFAULT_PROVIDER=libvirt
```
配置Vagrantfile：

```ruby
Vagrant.configure("2") do |config|
  config.vm.provider :libvirt do |libvirt|
    libvirt.host = "example.com"
  end
end
```

参考：

1. [Vagrant Official Site][vagrant2]
2. [Vagrant-Libvirt Plugin][vagrant1]

[vagrant1]: https://github.com/pradels/vagrant-libvirt/ "Vagrant-Libvirt"
[vagrant2]: https://docs.vagrantup.com/v2/ "Vagrant Official Site"
