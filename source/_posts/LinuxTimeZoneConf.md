layout: blog
title: "LinuxTimeZoneConf"
date: 2015-04-15 15:13:59
tags: [linux]
categories: [linux]
list_number: false

---

### 更改/etc/localtime

删除/etc/localtime：

```
$ rm -f /etc/localtime
```

比如选择中国上海时区，重新做个软链接：

```
$ ln -s /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
```

### 配置NTP服务器
修改/etc/ntp.conf：

```
...
restrict 10.0.100.0 mask 255.255.255.0 nomodify #允许某网段客户端访问
...
server s1a.time.edu.cn #使用教育网内时间源
server 127.127.1.0 ＃本地时钟，当断网时保证同步
fudge 127.127.1.0 stratum 10
...
```

启动ntpd服务：

```
$ service ntpd start
```

客户端同步时间：

```
$ ntpdate 10.0.100.6
```

可以用crontab定时同步，如每2小时同步一次，crontab -e：

```
0 */2 * * * /usr/sbin/ntpdate 10.0.100.6
```


参考：

1. [linux下超简单的ntp时间服务器][1]
2. [Linux下crontab的使用][2]

[1]: http://dngood.blog.51cto.com/446195/662451 "linux下超简单的ntp时间服务器"
[2]: http://yangqijun.iteye.com/blog/1173016 "Linux下crontab的使用"