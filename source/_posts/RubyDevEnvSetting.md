---
title: "Ruby开发环境搭建"
date: 2015-04-17 10:48:49
tags: [ruby, dev]
categories: [Tech]
---

### Ruby开发环境搭建

多版本管理`RVM`：

```bash
$ gpg --keyserver hkp://keys.gnupg.net --recv-keys 409B6B1796C275462A1703113804BB82D39DC0E3
$ curl -sSL https://get.rvm.io | bash -s stable
$ source /etc/profile.d/rvm.sh
$ sed -i 's!cache.ruby-lang.org/pub/ruby!ruby.taobao.org/mirrors/ruby!' $rvm_path/config/db # 使用taobao源下载ruby
$ rvm list known
$ rvm install 2.1.4
$ rvm docs generate-ri # 生成ruby文档
$ rvm use 2.1.4 --default # 设定默认ruby版本
$ rvm list # 查询已安装版本
```
用`gemset`建立独立环境：

```bash
$ rvm use 2.1.4
$ rvm gemset create rails42
$ rvm use 2.1.4@rails42
$ rvm gemset list
```

`Gem`管理ruby包：

```bash
$ gem sources --remove https://rubygems.org/
$ gem sources -a https://ruby.taobao.org/
$ gem sources -l
$ gem search rails # search，install, etc
```


参考：

1. [RVM official site][ruby1]
2. [RVM Guide from ruby-china.org][ruby2]
3. [Gem Official Guide][ruby3]
4. [Ruby@Taobao.org][ruby4]

[ruby1]: https://rvm.io/ "RVM official site"
[ruby2]: https://ruby-china.org/wiki/rvm-guide "RVM Guide from ruby-china.org"
[ruby3]: http://guides.rubygems.org/rubygems-basics/ "Gem Official Guide"
[ruby4]: http://ruby.taobao.org/ "Ruby@Taobao.org"
