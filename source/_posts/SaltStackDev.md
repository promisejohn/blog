---
title: "SaltStack开发环境搭建"
date: 2015-04-16 11:08:18
tags: [saltstack,dev]
categories: [Tech]
---

### 安装及准备
切换到python独立环境

```bash
$ pyenv virtualenv salt-2.7.8
```
获取代码：

```bash
$ git clone https://github.com/saltstack/salt
$ git fetch --tags 
```

安装：

```bash
$ env SWIG_FEATURES="-cpperraswarn -includeall -D__`uname -m`__ -I/usr/include/openssl" pip install M2Crypto
$ pip install pyzmq PyYAML pycrypto msgpack-python jinja2 psutil
$ pip install -e ./salt
```

配置：

```
$ mkdir -p $(dirname `pyenv which python`)/../etc/salt
$ cp ./salt/conf/master ./salt/conf/minion $(dirname `pyenv which python`)/../etc/salt
```
master配置文件：

1. user: root
2. root_dir: $(dirname `pyenv which python`)/..
3. pidfile: $(dirname `pyenv which python`)/../salt-master.pid
4. publish_port: 14505
5. ret_port: 14506

minion配置文件：

1. user: root
2. root_dir: $(dirname `pyenv which python`)/..
3. pidfile: $(dirname `pyenv which python`)/../salt-minion.pid
4. master: localhost
5. id: saltdev
6. master_port: 14505

运行启动：

```bash
$ cd $(dirname `pyenv which python`)/../
$ salt-master -c ./etc/salt -d
$ salt-minion -c ./etc/salt -d
$ salt-key -c ./etc/salt -L
$ salt-key -c ./etc/salt -A
$ salt -c ./etc/salt '*' test.ping
```

其他：
1. 通过`-l debug`开启debug模式，去掉`-d`直接输出到console。
2. socket path在linux上最多107个字符，可以通过缩短sock_dir和root_dir字符。
3. `ulimit -n`检查File descriptor limites，至少2047：`ulimit -n 2048`

### 文档生成

```bash
$ pip install Sphinx==1.3b2
$ cd doc; make html
$ cd _build/html; python -m SimpleHTTPServer
```

### Tests

```bash
$ ./setup.py test
```


#### 参考：
1. [saltstack官方文档][python1]

[python1]: http://docs.saltstack.com/en/latest/topics/development/hacking.html "saltstack官方文档"
