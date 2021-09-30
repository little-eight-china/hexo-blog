---
title: Docker简单介绍
date: 2019-02-27 21:38:05
categories: 
  - docker
tags: 
  - little_eight
---

# Docker

* 镜像（ Image ）
* 容器（ Container ）
* 仓库（ Repository ）

## Docker镜像
Docker 镜像（Image），就相当于是一个 root 文件系统，它除了提供容器运行时所需的程序、库、资
源、配置等文件外，还包含了一些为运行时准备的一些配置参数（如匿名卷、环境变量、用户等）。镜像不包含任何动态数据，其内容在构建之后也不会被改变。

### 分层存储
因为镜像体积庞大，就将其设计成分层存储的架构，所以严格来说镜像由一组文件系统组成。
镜像构建时，会一层层构建，前一层是后一层的基础。。每一层构建完就不会再发生改变，后一层上的任何改变只发生在自己这一层。比如，删除前一层文件的操作，实际不是真的删除前一层的文件，而是仅在当前层标记为该文件已删除。在最终容器运行的时候，虽然不会看到这个文件，但是实际上该文件会一直跟随镜像。
<!--more--> 
## Docker 容器
容器是镜像运行时的实体。容器可以被创建、启动、停止、删除、暂停等。

## Docker仓库
镜像构建完成后，可以很容易的在当前宿主机上运行，但是，如果需要在其它服务器上使用这个镜像，我们就需要一个集中的存储、分发镜像的服务，Docker仓库就是这样的服务。
一个 Docker Registry 中可以包含多个仓库（ Repository ）；每个仓库可以包含多个标签（ Tag ）；每个标签对应一个镜像。
通常，一个仓库会包含同一个软件不同版本的镜像，而标签就常用于对应该软件的各个版本。我们可以通过 <仓库名>:<标签> 的格式来指定具体是这个软件哪个版本的镜像。如果不给出标签，将以 latest 作为默认标签。
以 Ubuntu 镜像 为例， ubuntu 是仓库的名字，其内包含有不同的版本标签，如， 16.04 , 18.04。我们可以通过 ubuntu:14.04，或者ubuntu:18.04来具体指定所需哪个版本的镜像。如果忽略了标签，比如 ubuntu ，那将视为ubuntu:latest 。

### 私有 Docker仓库
除了使用公开服务外，用户还可以在本地搭建私有Docker仓库。Docker官方提供了Docker仓库镜像，可以直接使用做为私有 Registry 服务。

## 使用 Docker 镜像

### 获取镜像
从 Docker 镜像仓库获取镜像的命令是 docker pull 。其命令格式为：
> docker pull [选项] [Docker Registry 地址[:端口号]/]仓库名[:标签]

* Docker 镜像仓库地址：地址的格式一般是 <域名/IP>[:端口号] 。默认地址是 Docker Hub。
* 仓库名：如之前所说，这里的仓库名是两段式名称，即<用户名>/<软件名>。对于DockerHub，如果不给出用户名，则默认为 library ，也就是官方镜像。
比如
<!--more-->
> $ docker pull ubuntu:18.04
18.04: Pulling from library/ubuntu
bf5d46315322: Pull complete
9f13e0ac480c: Pull complete
e8988b5b3097: Pull complete
40af181810e7: Pull complete
e6f7c7e5c03e: Pull complete
Digest: sha256:147913621d9cdea08853f6ba9116c2e27a3ceffecf3b49298
3ae97c3d643fbbe
Status: Downloaded newer image for ubuntu:18.04

上面的命令中没有给出 Docker 镜像仓库地址，因此将会从 Docker Hub 获取镜像。而镜像名称是 ubuntu:18.04 ，因此将会获取官方镜像 library/ubuntu仓库中标签为 18.04 的镜像。

### 运行

有了镜像后，我们就能够以这个镜像为基础启动并运行一个容器。以上面的
ubuntu:18.04 为例，如果我们打算启动里面的 bash 并且进行交互式操作的
话，可以执行下面的命令。

> $ docker run -it --rm \
ubuntu:18.04 \
bash
root@e7009c6ce357:/# cat /etc/os-release
NAME="Ubuntu"
VERSION="18.04.1 LTS (Bionic Beaver)"
ID=ubuntu
ID_LIKE=debian
PRETTY_NAME="Ubuntu 18.04.1 LTS"
VERSION_ID="18.04"
HOME_URL="https://www.ubuntu.com/"
SUPPORT_URL="https://help.ubuntu.com/"
BUG_REPORT_URL="https://bugs.launchpad.net/ubuntu/"
PRIVACY_POLICY_URL="https://www.ubuntu.com/legal/terms-and-polic
ies/privacy-policy"
VERSION_CODENAME=bionic
UBUNTU_CODENAME=bionic

* docker run 就是运行容器的命令
* -it ：这是两个参数，一个是 -i ：交互式操作，一个是 -t 终端。我们
这里打算进入 bash 执行一些命令并查看返回结果，因此我们需要交互式终
端。
* --rm ：这个参数是说容器退出后随之将其删除。默认情况下，为了排障需
求，退出的容器并不会立即删除，除非手动 docker rm 。我们这里只是随便
执行个命令，看看结果，不需要排障和保留结果，因此使用 --rm 可以避免
浪费空间。
* ubuntu:18.04 ：这是指用 ubuntu:18.04 镜像为基础来启动容器。
* bash ：放在镜像名后的是命令，这里我们希望有个交互式 Shell，因此用的
是 bash 。

### 列出镜像
要想列出已经下载下来的镜像，可以使用 docker image ls 命令。
> $ docker image ls
REPOSITORY TAG IMAGE ID CRE
ATED SIZE
redis latest 5f515359c7f8 5 d
ays ago 183 MB
nginx latest 05a60462f8ba 5 d
ays ago 181 MB
mongo 3.2 fe9198c04d62 5 d
ays ago 342 MB
<none> <none> 00285df0df87 5 d
ays ago 342 MB
ubuntu 18.04 f753707788c5 4 w
eeks ago 127 MB
ubuntu latest f753707788c5 4 w
eeks ago 127 MB

列表包含了 仓库名 、 标签 、 镜像 ID 、 创建时间 以及 所占用的空间 。

### 删除本地镜像
如果要删除本地的镜像，可以使用 docker image rm 命令

## 使用 Dockerfile 定制镜像

Dockerfile 是一个文本文件，其内包含了一条条的指令(Instruction)，每一条指令
构建一层，因此每一条指令的内容，就是描述该层应当如何构建。

看一个Dockerfile文件
```xml
FROM nginx
RUN echo '<h1>Hello, Docker!</h1>' > /usr/share/nginx/html/index
.html
```

### FROM 指定基础镜像

所谓定制镜像，那一定是以一个镜像为基础，在其上进行定制。就像我们之前运行
了一个 nginx 镜像的容器，再进行修改一样，基础镜像是必须指定的。而
FROM 就是指定基础镜像，因此一个 Dockerfile 中 FROM 是必备的指令，并
且必须是第一条指令。

### RUN 执行命令
RUN 指令是用来执行命令行命令的。由于命令行的强大能力， RUN 指令在定制
镜像时是最常用的指令之一。其格式有两种：

* shell 格式： RUN <命令> ，就像直接在命令行中输入的命令一样。刚才写的
Dockerfile 中的 RUN 指令就是这种格式。
```
RUN echo '<h1>Hello, Docker!</h1>' > /usr/share/nginx/html/index
.html
```
* exec 格式： RUN ["可执行文件", "参数1", "参数2"] ，这更像是函数调用中
的格式。

Dockerfile 支持 Shell 类的行尾添加 \ 的
命令换行方式，以及行首 # 进行注释的格式。良好的格式，比如换行、缩进、注
释等，会让维护、排障更为容易，这是一个比较好的习惯。

### 构建镜像

在 Dockerfile 文件所在目录执行：
```
$ docker build -t nginx:v3 .
Sending build context to Docker daemon 2.048 kB
Step 1 : FROM nginx
---> e43d811ce2f4
Step 2 : RUN echo '<h1>Hello, Docker!</h1>' > /usr/share/nginx/h
tml/index.html
---> Running in 9cdc27646c7b
---> 44aa4490ce2c
Removing intermediate container 9cdc27646c7b
Successfully built 44aa4490ce2c

```

这里我们使用了 docker build 命令进行镜像构建。其格式为：
> docker build [选项] <上下文路径/URL/->

在这里我们指定了最终镜像的名称 -t nginx:v3 ，构建成功后，我们可以像之前
运行 nginx:v2 那样来运行这个镜像，其结果会和 nginx:v2 一样。

更多Dockerfile命令可参考官方文档：https://docs.docker.com/engine/reference/builder/



## docker使用

### docker安装

* 使用 sudo 或 root 权限登录 Centos，确保 yum 包更新到最新。
```
$ sudo yum update
```

* 执行 Docker 安装脚本。
```
$ curl -fsSL https://get.docker.com -o get-docker.sh
$ sudo sh get-docker.sh
```
* 启动 Docker 进程。
```
sudo systemctl start docker
```


后期可能会出现这样的错误信息：
>Got permission denied while trying to connect to the Docker daemon socket at unix:///var/run/docker.sock: Get http://%2Fvar%2Frun%2Fdocker.sock/v1.39/images/json: dial unix /var/run/docker.sock: connect: permission denied

原因：
>docker守护进程启动的时候，会默认赋予名字为docker的用户组读写Unix socket的权限，因此只要创建docker用户组，并将当前用户加入到docker用户组中，那么当前用户就有权限访问Unix socket了，进而也就可以执行docker相关命令。

解决方案
> .#添加docker用户组
sudo groupadd docker    
 #添加将登陆用户加入到docker用户组中
sudo gpasswd -a $USER docker    
 #将登陆用户加入到docker更新用户组中
newgrp docker     #更新用户组
docker ps   
 #测试docker命令是否可以使用sudo正常使用


## Kubernetes（太多，后期再补充吧）

Kubernetes，因为首尾字母中间有8个字符，所以被简写成 K8s。


