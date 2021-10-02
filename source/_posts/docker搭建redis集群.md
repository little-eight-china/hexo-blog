---
title: docker搭建redis集群
date: 2019-06-30 18:18:42
categories: 
  - 操作实战
tags: 
  - little_eight
---
## 拉取redis镜像（版本5.0.5）
`docker pull redis:5.0.5`
![](https://gitee.com/littleeight/blog-images/raw/master/docker%E6%90%AD%E5%BB%BAredis%E9%9B%86%E7%BE%A4/1.png)

## 创建并运行三个redis容器

- redis-node1 6379
- redis-node1 6380
- redis-node1 6381

<!--more--> 

`docker run -d --name redis-node1 -v /data/redis-data/node1:/data -p 6379:6379 redis:5.0.5 --cluster-enabled yes --cluster-config-file nodes-node-1.conf`
​

`docker run -d --name redis-node2 -v /data/redis-data/node2:/data -``p 6380:6379 redis:5.0.5 --cluster-enabled yes --cluster-config-file nodes-node-2.conf`
​

`docker run -d --name redis-node3 -v /data/redis-data/node3:/data -p 6381:6379 redis:5.0.5 --cluster-enabled yes --cluster-config-file nodes-node-3.conf`
​

## 查看三个容器是否正常运行
`docker ps`
![](https://gitee.com/littleeight/blog-images/raw/master/docker%E6%90%AD%E5%BB%BAredis%E9%9B%86%E7%BE%A4/2.png)

## 查看容器分配的ip
`docker inspect redis-node1 | grep IPAddress`
`docker inspect redis-node2 | grep IPAddress`
`docker inspect redis-node3 | grep IPAddress`
​
![](https://gitee.com/littleeight/blog-images/raw/master/docker%E6%90%AD%E5%BB%BAredis%E9%9B%86%E7%BE%A4/3.png)

&ensp;&ensp;&ensp;&ensp;可以得到

- redis-node1 172.18.0.2
- redis-node2 172.18.0.3
- redis-node3 172.18.0.4

​

## 进入容器配置集群信息
### 进入容器
`docker exec -it redis-node1 /bin/bash`
### 使用redis-cli连接配置（注意ip需要替换成自己的）
`redis-cli --cluster create 172.18.0.2:6379  172.18.0.3:6379 172.18.0.4:6379 --cluster-replicas 0`
&ensp;&ensp;&ensp;&ensp;中途记得输入yes，回车的话是没有默认值的。
![](https://gitee.com/littleeight/blog-images/raw/master/docker%E6%90%AD%E5%BB%BAredis%E9%9B%86%E7%BE%A4/4.png)

### 查看集群信息
`cluster nodes`
`cluster info`
&ensp;&ensp;&ensp;&ensp;状态是ok，说明集群正常。
![](https://gitee.com/littleeight/blog-images/raw/master/docker%E6%90%AD%E5%BB%BAredis%E9%9B%86%E7%BE%A4/5.png)

&ensp;&ensp;&ensp;&ensp;至此，集群搭建完毕。
​

## 测试集群
&ensp;&ensp;&ensp;&ensp;使用`redis-cli -c` 连接到集群节点，然后set值，set值之后会根据hash槽算法存储到指定的节点。现在有三个节点，则node1分配槽为0－5460，node2分配槽为5461－10922，node3分配槽为10923－16383。现在测试set key值123跟1234567890。可以发现不同的key会存到不同的节点。
​

![](https://gitee.com/littleeight/blog-images/raw/master/docker%E6%90%AD%E5%BB%BAredis%E9%9B%86%E7%BE%A4/6.png)


&ensp;&ensp;&ensp;&ensp;用rdm可看到set的俩值在不同的节点。
![](https://gitee.com/littleeight/blog-images/raw/master/docker%E6%90%AD%E5%BB%BAredis%E9%9B%86%E7%BE%A4/7.png)

&ensp;&ensp;&ensp;&ensp;至此，成功搭建了集群。




