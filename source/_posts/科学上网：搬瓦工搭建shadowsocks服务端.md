---
title: 科学上网：搬瓦工VPS搭建shadowsocks
date: 2018-08-21 23:58:13
categories: 
  - 网络
tags: 
  - cz_wsd
---

### 前言
Shadowsocks，中文名影梭，使用socks5代理，在中国大陆广泛用于翻墙，速度比 pptp和OpenVPN 要快，是一款翻墙必备神器。

### VPS
Shadowsocks的正常使用需要服务器端，其实，所有的翻墙软件都是通过服务器端，而搭建服务器端，你就需要有自己的VPS，所以第一步你就是需要购买一个自己的VPS(或者你可以跟别人合租)，现在普遍使用的搭建服务器端的vps主要包括3种，一个是Linode，一个是DigitalOcean，一个是BandwagonHOST(搬瓦工)，这是从价格，性能等方面做出的推荐。

### 搬瓦工
搬瓦工VPS是一款性价比较高的便宜VPS主机，且适合入门级网友学习Linux和建站用途。（具体可在[搬瓦工VPS中文网](http://banwagong.cn/)查看） 
一、选择对应且需要的VPS方案
![这里写图片描述](https://img-blog.csdn.net/20180821233117700?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2N6X3dzZA==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)
<!--more-->
二、核对方案配置以及选择时间期限和机房
![这里写图片描述](http://banwagong.cn/images/jiaocheng-1.jpg)
然后会看到上图所示的界面，根据图示我们需要确定选择的方案以及时间，默认是洛杉矶机房，我们也可以选择其他的几个机房之一，点击"ADD TO CART"添加购物车。
![这里写图片描述](http://banwagong.cn/images/jiaocheng-2.jpg)

> 搬瓦工优惠码：当前我们可以使用 BWH26FXH3HIQ 优惠码再节省5%

三、登录或者新注册搬瓦工账户
![这里写图片描述](http://banwagong.cn/images/jiaocheng-3.jpg)
根据上图，如果我们有过账户，可以直接点击"Click here to login"登录以及付款就可以，如果还没有账户则需要注册账户。个人信息不要真实的，但也不能太离谱和乱写字符出来，好歹也要稍微用点拼音。

> 我们不能用代理IP登录注册账户，国家需要真实，不要乱选择。

四、付款成功以及使用
目前只支持Paypal付款，所以我们需要有账户，如果没有可以注册，且可以用账户余额，也可以绑定信用卡、银联支付
![这里写图片描述](http://banwagong.cn/images/jiaocheng-4.jpg)
稍等一分钟左右，我们登录搬瓦工账户后台，就可以看到已经成功后买到的VPS主机，我们可以登录面板使用。
### 使用
点击 “KiwiVM Control Panel” 进入 VPS 控制面板。
![这里写图片描述](http://s15.sinaimg.cn/mw690/006QTR5ezy7htKLALh4ae&690)
接下来，使用xshell进行连接。

> 温馨提示：如果出现死活连接不上linode的情况，可以ping一下你的Ip，看是否能ping通，如果ping不通，可能是你的ip已经被墙了，建议你删除此vps，重新建一个，或者在Linode中申请更换Ip的工单！

### Shadowsocks服务端搭建
1.环境安装与更新
这一步，需要做的是依次执行下面的每条命令：
> yum install epel-release
yum update
yum install python-setuptools m2crypto supervisor
easy_install pip
pip install shadowsocks

2.文件配置
接下来需要编辑一下/etc/shadowsocks.json文件，命令如下：
> vi /etc/shadowsocks.json

执行上述命令后，此时的你已经进入文件编辑模式，这是你创建的一个新的空白文件，你需要做的事情就是将下面的内容粘贴后复制到shadowsocks.json文件里：

###### 单用户配置：
> {
    "server":"your_server_ip", 
    "server_port":8388,
    "local_port":1080,
    "password":"yourpassword",
    "timeout":600,
    "method":"aes-256-cfb"
}

各字段的含义： 
server：服务器 IP (IPv4/IPv6)，注意这也将是服务端监听的 IP 地址 
server_port：监听的服务器端口 
local_address：本地监听的 IP 地址 
local_port：本地端端口 
password：用来加密的密码 
timeout：超时时间（秒） 
method：加密方法，可选择 “bf-cfb”, “aes-256-cfb”, “des-cfb”, “rc4”, 等等。默认是一种不安全的加密，推荐用 “aes-256-cfb” 
fast_open：true 或 false 
works：works数量，默认为 1
###### 多用户配置：

> {
 "server":"my_server_ip"，
 "local_address": "127.0.0.1",
 "local_port":1080,
  "port_password": {
     "8381": "foobar1",
     "8382": "foobar2",
     "8383": "foobar3",
     "8384": "foobar4"
 },
 "timeout":300,
 "method":"aes-256-cfb",
 "fast_open": false
}
###### 启动
> ssserver -c /etc/shadowsocks.json -d start #启动 
ssserver -c /etc/shadowsocks.json -d stop #停止
###### 启动
日志文件路径：/var/log/shadowsocks.log

### 配置自动启动
编辑/etc/supervisord.conf文件，命令如下：

> vi /etc/supervisord.conf

 此时，你已进入supervisord.conf文件的编辑模式，这不是一个空白文件，里面有很多英文，请把下面的内容粘贴到文件尾部的空行处，然后保存：
> [program:shadowsocks]
command=ssserver -c /etc/shadowsocks.json
autostart=true
autorestart=true
user=root
log_stderr=true
logfile=/var/log/shadowsocks.log

接下来需要编辑/etc/rc.local文件，请执行以下命令：

> vi /etc/rc.local

此时，进入了rc.local文件的编辑模式，这也不是一个空白文件，请把以下内容粘贴到文件中部的空白处，然后保存

> service supervisord start

最后执行reboot命令或者vps的重启按钮，重启服务器。

> 小提醒：搬瓦工的VPS在执行完reboot命令后有时会遇到重启失败的情况，这时候进入控制面板，看一下“Status”是不是“Running”，如果不是的话，点一下“Actions”里的“start”按钮即可。

最后，可通过ps -ef|grep shadowsocks查看shadowsocks进程是否已启动
