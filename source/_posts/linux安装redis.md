---
title: linux安装redis
date: 2019-04-15 22:15:27
categories: 
  - 操作实战
tags: 
  - little_eight
---

## 假设没有虚拟机！
[vm安装centos7](https://blog.csdn.net/babyxue/article/details/80970526)

## redis的操作

* 下载忽略，下载到/usr/local
* 进到redis根目录，yum install gcc，下载gcc，然后make MALLOC=libc
* 进入src, make install
* 回到根目录，vi redis.conf
>1、注释bind 127.0.0.1或一些bind相关的
2、修改protected-mode=no，开放外界访问redis
3、daemonize属性改为yes，表明需要在后台运行

* 防火墙的处理
>停止使用firewall
systemctl stop firewalld.service
禁止在开机启动
systemctl disable firewalld.service docker ps   

* 安装结束
