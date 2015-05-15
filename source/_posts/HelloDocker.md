---
title: "HelloDocker"
date: 2015-05-13 15:56:19
tags: [docker]
categories: [Tech]
---

# Docker概述

Docker是基于Go语言开发的容器管理工具，而且抽象级别比lxc这些管理工具高，目前官方默认通过libcontainer管理容器。lxc是一些kernel patch（namespaces）和userspace tool（cgroup）的集合，通过cgroup这个用户态管理工具实现对隔离资源的管理。Docker相比LXC增加了镜像服务（通过UnionFS和DeviceMapper），同时简化了操作复杂度，可以很方便实现应用分发部署，甚至是扩容。目前也有一些周边管理工具（Fig、Flocker、weaver）在创新，在docker基础上实现更多实用功能，如迁移，流程编排，网络管理等。跟openstack相比，docker更年轻，也更轻量级，但两者在某些场景下又可以很好结合起来，比如通过docker将openstack的管理节点实现高可用、可扩展。

docker的部署在发行版上比较简单，但对内核版本有不同要求，比如在centos6.5上要求内核版本至少2.6.32-431。

```bash
$ yum install -y docker-io
$ chkconfig docker on
$ service docker start
$ docker run -it ubuntu /bin/bash
$ docker info #查看docker信息
```

## Docker基本用法

### 基本命令使用：

```bash
$ docker run ubuntu:14.04 /bin/echo 'Hello world'
$ docker run -d ubuntu:14.04 /bin/sh -c "while true; do echo hello world; sleep 1; done" # 后台运行
$ docker ps -a
$ docker logs instance_name
$ docker logs -f instance_name # 持续输出
$ docker stop/start/restart instance_name # stop容器后还能启动，但网络信息会变
$ docker ps -aq | xargs docker rm # 删除所有容器，运行中的会保留
$ docker images ＃ -tree参数可以看层级关系，最大128层
$ docker save -o ubuntu.tar ubuntu:14.04 # 把镜像保存到本地
$ tar -tf ubuntu.tar # 可以看到image的结构，包含了多层FS
$ docker rmi ubuntu
$ docker run -d -P training/webapp python app.py # 映射所有端口
$ docker port instance_name 5000 #查看被映射的端口
$ docker run -d -p 5000:5000 training/webapp python app.py # 映射5000端口
$ docker run -d -p 127.0.0.1:5000:5000/udp training/webapp python app.py # 映射本机UDP端口
$ docker top instance_name
$ docker inspect instance_name # 查看容器配置
$ docker inspect -f '{{ .NetworkSettings.IPAddress }}' instance_name # 查看具体的配置信息，如IP
$ docker pull centos
$ docker commit -m "some modify hints" -a "author info" 
contain_id repo_name/image_name:tags
$ docker tag container_id repo_name/image_name:tags
$ docker exec -it CONTAINER_NAME /bin/bash # 1.3开始支持在运行的容器内执行命令
```

### 使用Dockerfile创建image

`mkdir myimage && cd $_ && vim Dockerfile`:

```
# This is a comment
FROM ubuntu:14.04
MAINTAINER Promise John <promise.jon@gmail.com>
RUN apt-get update && apt-get install -y nginx
```

### 使用Linking

Docker主要通过环境变量和`/etc/hosts`文件来提供访问途径：

```bash
$ docker run -d --name db training/postgres
$ docker run -d -P --name web --link db:db training/webapp python app.py
$ docker inspect -f "{{ .Name }}" instance_id # 查看容器名
$ docker inspect -f "{{ .HostConfig.Links }}" web
$ docker run --rm --name web2 --link db:db training/webapp env
$ docker run --rm --name web2 --link db:db training/webapp cat /etc/hosts
```

### Storage
主要包括独立的卷和共享本地文件系统：

```bash
$ docker run -d -P --name web -v /webapp training/webapp python app.py # volumn不会随容器消失，作为数据独立存在
$ docker inspect web # 查看/var/lib/docker/volumes和vfs
$ docker run -d -P --name web -v /src/webapp:/opt/webapp training/webapp python app.py # 挂载
$ docker run -d -P --name web -v /src/webapp:/opt/webapp:ro training/webapp python app.py # 只读挂载
$ docker run --rm -it -v ~/.bash_history:/.bash_history ubuntu /bin/bash # 只挂载某个文件
```

Container间共享数据：

```bash
$ docker create -v /dbdata --name dbdata training/postgres /bin/true
$ docker run -d --volumes-from dbdata --name db1 training/postgres
$ docker run -d --volumes-from dbdata --name db2 training/postgres
$ docker run --volumes-from dbdata -v $(pwd):/backup ubuntu tar cvf /backup/backup.tar /dbdata # 备份容器数据
$ docker run --volumes-from dbdata2 -v $(pwd):/backup busybox tar xvf /backup/backup.tar # 恢复数据
```


## Docker本地仓库

Docker Hub基本用法：

```bash
$ docker search ubuntu
$ docker pull ubuntu:14.04
$ docker login # 配置文件在~/.dockercfg
$ docker push yourname/newimage
```

配置本地的registry：

```bash
$ docker run -d -p 5000:5000 registry
$ docker tag ubuntu:14.04 localhost:5000/ubuntu:14.04
$ docker push localhost:5000/ubuntu:14.04
$ curl -v http://localhost:5000/v1/repositories/ubuntu/tags/14.04 # 查看image id
$ docker run -d -p 5000:5000 \
    -e STANDALONE=false \
    -e MIRROR_SOURCE=https://registry-1.docker.io \
    -e MIRROR_SOURCE_INDEX=https://index.docker.io \
    registry # 作为官网镜像启动，代理模式
```

使用本地的registry：

可以使用`--registry-mirror`参数启动：

```bash
$ docker --registry-mirror=http://10.101.29.26 -d
```


## 小结

Docker在快速应用分发、扩展等方面有很高的自动化能力，但是跟主机虚拟化一样，网络也是需要重点考虑的难点。默认情况下都是通过本地的一个bridge提供容器之间的通信，通过NAT为应用提供地址，一般的应用部署足以应付，但如果要实现更复杂的网络场景，如多主机上container组成大规模的L3负载均衡集群、应用无缝迁移等，还是需要大量的定制，目前这方面的周边产品开发比较活跃。


## 参考
1. [Official Docs][docker0]
2. [Docker with aufs and devicemapper][docker1]
3. [LXC Tools][docker2]


[docker0]:https://docs.docker.com "Official Docs"
[docker1]:http://www.infoq.com/cn/articles/analysis-of-docker-file-system-aufs-and-devicemapper "Docker fs with aufs and devicemapper"
[docker2]:http://www.ibm.com/developerworks/cn/linux/l-lxc-containers/ "LXC TOOL"