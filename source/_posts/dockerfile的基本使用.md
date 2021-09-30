---
title: dockerfile的基本使用
date: 2019-08-29 16:55:00
categories: 
  - docker
tags: 
  - little_eight
---

## dockerfile的属性
​

### FROM 
基础镜像，该配置是基于某些镜像的基础上实现的，比如jdk镜像


### MAINTAINER 
维护者信息


### ENV
设置环境变量


### ADD
文件放在当前目录下，拷过去会自动解压


### RUN
执行命令。RUN命令是创建Docker镜像（image）的步骤，RUN命令对Docker容器造成的改变是会被反映到创建的Docker镜像上的。一个Dockerfile中可以有许多个RUN命令。


### WORKDIR
指定容器的一个目录， 容器启动时执行的命令会在该目录下执行


### EXPOSE
映射端口


### CMD
镜像最终运行的命令。CMD命令是当Docker镜像被启动后Docker容器将会默认执行的命令。一个Dockerfile中只能有一个CMD命令。通过执行docker run xxx启动镜像可以重载CMD命令。
​
<!--more--> 
## dockerfile例子
比如我们想运行一个jar包，先拉取jdk镜像
`docker pull openjdk:8`
​

然后创建一个dockerfile
`touch Dockerfile`
​

编辑这个文件


`FROM openjdk:8`
`MAINTAINER huangjuguan@gmail.com `
`ADD *.jar / `
`EXPOSE 80 `
`WORKDIR / `
`CMD ["bash", "-c", "java -jar *.jar"]`
​

## dockerfile使用
使用docker build的命令来构建镜像
`docker build -f Dockerfile -t dockerfile-test:v1 .`
​

docker build的命令大概如下：

- **--build-arg=[] :**设置镜像创建时的变量；
- **--cpu-shares :**设置 cpu 使用权重；
- **--cpu-period :**限制 CPU CFS周期；
- **--cpu-quota :**限制 CPU CFS配额；
- **--cpuset-cpus :**指定使用的CPU id；
- **--cpuset-mems :**指定使用的内存 id；
- **--disable-content-trust :**忽略校验，默认开启；
- **-f :**指定要使用的Dockerfile路径；
- **--force-rm :**设置镜像过程中删除中间容器；
- **--isolation :**使用容器隔离技术；
- **--label=[] :**设置镜像使用的元数据；
- **-m :**设置内存最大值；
- **--memory-swap :**设置Swap的最大值为内存+swap，"-1"表示不限swap；
- **--no-cache :**创建镜像的过程不使用缓存；
- **--pull :**尝试去更新镜像的新版本；
- **--quiet, -q :**安静模式，成功后只输出镜像 ID；
- **--rm :**设置镜像成功后删除中间容器；
- **--shm-size :**设置/dev/shm的大小，默认值是64M；
- **--ulimit :**Ulimit配置。
- **--squash :**将 Dockerfile 中所有的操作压缩为一层。
- **--tag, -t:** 镜像的名字及标签，通常 name:tag 或者 name 格式；可以在一次构建中为一个镜像设置多个标签。
- **--network:** 默认 default。在构建期间设置RUN指令的网络模式

​

使用docker images便可查看我们刚刚构建的镜像，然后就可以启动他了。
​

​
​

​

