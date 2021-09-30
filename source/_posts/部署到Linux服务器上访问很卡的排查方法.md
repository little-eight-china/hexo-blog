---
title: 部署到Linux服务器上访问很卡的排查方法
date: 2019-04-15 22:12:41
categories: 
  - linux
tags: 
  - little_eight
---

## 查看内存使用情况

free -g


当观察到free栏已为0的时候，表示内存基本被吃完了，那就释放内存吧



## 查看磁盘使用情况

df -h



当发现磁盘使用率很高时，那就要释放磁盘空间了，删除一些不必要的文件（查看各个目录占用磁盘空间，参考之前的du命令文章）

<!--more-->

## 查看磁盘IO使用情况

iostat -x 1

1表示1秒刷新一次


当发现最右侧%util很高时，表示IO就很高了，若想看哪个进程占用IO，执行iotop命令查看

你也可以测试自己的读写速率多大
```
dd if=/dev/zero of=test bs=64k count=4k oflag=dsync
```
输出的依次是:复制的大小、用时跟写速率


## 查看cpu使用情况

top

下图中红框里表是cpu使用情况，最右侧的%id表示剩余，若很低，则表示cpu被吃完了，在top界面按shift+p对进程使用cpu排序，能看到哪些进程占用cpu较多

（暂时没图、、）
