---
title: docker搭建mongodb
date: 2019-08-05 18:53:00
categories: 
  - 操作实战
tags: 
  - little_eight
---

## 下载与安装
&ensp;&ensp;&ensp;&ensp;拉取4.4.8版本的镜像，看仓库的描述是一百多兆的，没想到要400多。
`docker pull mongo:4.4.8`
![](https://gitee.com/littleeight/blog-images/raw/master/docker%E6%90%AD%E5%BB%BAmongodb/1.png)

## 运行容器
&ensp;&ensp;&ensp;&ensp;mongodb默认是没有用户验证的，所以加个--auth开启，并把数据持久化到/data/mongo-data中
`docker run --name mongo -v /data/mongo-data:/data/db -p 27017:27017 -d mongo:4.4.8 --auth`
&ensp;&ensp;&ensp;&ensp;**进入admin数据库**
`docker exec -it mongo mongo admin`

<!--more--> 
&ensp;&ensp;&ensp;&ensp;**创建用户时需要了解的属性**

1. user：用户名
1. pwd：密码
1. roles：指定用户的角色，可以用一个空数组给新用户设定空角色；在roles字段,可以指定内置角色和用户定义的角色。

     role ：角色可以选：
1.数据库用户角色：read、readWrite;
2.数据库管理角色：dbAdmin、dbOwner、userAdmin；
       3. 集群管理角色：clusterAdmin、clusterManager、clusterMonitor、hostManager；
       4. 备份恢复角色：backup、restore；
       5. 所有数据库角色：readAnyDatabase、readWriteAnyDatabase、userAdminAnyDatabase、dbAdminAnyDatabase
       6. 超级用户角色：root 
    // 这里还有几个角色间接或直接提供了系统超级用户的访问（dbOwner 、userAdmin、userAdminAnyDatabase）
       7. 内部角色：__system
db：指定使用的数据库
​

&ensp;&ensp;&ensp;&ensp;其中具体角色的意思：
Read：允许用户读取指定数据库
readWrite：允许用户读写指定数据库
dbAdmin：允许用户在指定数据库中执行管理函数，如索引创建、删除，查看统计或访问system.profile
userAdmin：允许用户向system.users集合写入，可以找指定数据库里创建、删除和管理用户
clusterAdmin：只在admin数据库中可用，赋予用户所有分片和复制集相关函数的管理权限。
readAnyDatabase：只在admin数据库中可用，赋予用户所有数据库的读权限
readWriteAnyDatabase：只在admin数据库中可用，赋予用户所有数据库的读写权限
userAdminAnyDatabase：只在admin数据库中可用，赋予用户所有数据库的userAdmin权限
dbAdminAnyDatabase：只在admin数据库中可用，赋予用户所有数据库的dbAdmin权限。
root：只在admin数据库中可用。超级账号，超级权限。
​

**创建一个名为admin，密码是12345的超管用户**

`db.createUser({user:"admin",pwd:"12345",roles:[{role:'root',db:'admin'}]})`
![](https://gitee.com/littleeight/blog-images/raw/master/docker%E6%90%AD%E5%BB%BAmongodb/2.png)


## 测试新建用户的连接
`db.auth('admin','12345')`
&ensp;&ensp;&ensp;&ensp;连接成功会返回一个1
![](https://gitee.com/littleeight/blog-images/raw/master/docker%E6%90%AD%E5%BB%BAmongodb/3.png)





​

​

