---
title: Github Page+Hexo搭建个人博客
date: 2018-07-16 21:58:09
categories: 
  - blog
tags: 
  - little_eight
---

简单介绍怎么搭起这个博客的

### 首先执行四条命令（[Hexo官网](https://hexo.io/zh-cn)）

``` bash
$ npm install hexo-cli -g
$ hexo init blog
$ cd blog
$ npm install
$ hexo server
```
1、打开你的localhost:4000便可看到初始化的页面

### 然后去下主题（现在用的是nexT）
``` bash
git clone https://github.com/theme-next/hexo-theme-next.git
```
1、下完主题，把整个hexo-theme-next的文件夹拿到themes包下

2、修改根目录的配置_config.yml，把theme: 后面的改成hexo-theme-next
``` bash
theme: hexo-theme-next
```
<!--more-->
3、再执行命令
``` bash
$ hexo clean
$ hexo g
$ hexo s
```
4、打开你的localhost:4000，是不是变化了，当然还是英文的，如果你想主题为中文，改根目录配置_config.yml
``` bash
language: zh-cn
``` 
### 上传到github
1、在自己的github上面新建一个repository,然后在repository name里输入你的 “用户名+.github.io”，create

2、在配置_config.yml修改对应的参数
``` bash
deploy:
  type: git
  repository: git@github.com:xx/xx.github.io.git
  branch: master
```
3、如果你是第一次装git，搞定你的ssh问题（[随便百度的教材](https://www.cnblogs.com/ayseeing/p/3572582.html)）

3.5、如果步骤4会报错的话，就先装下git的插件。
``` bash
npm install hexo-deployer-git --save
```
4、运行命令吧
``` bash
hexo d
```
5、等它部署完成，访问你的地址
``` bash
https://xx.github.io/
```

### 当然你也可以搞成自动化，参照这个
[使用 Travis 自动部署 Hexo 到 Github 与 自己的服务器](https://segmentfault.com/a/1190000009054888)
