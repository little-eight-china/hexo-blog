---
title: Linux的基本命令入门
date: 2018-09-27 20:30:42
categories: 
  - linux
tags: 
  - little_eight
---

# Linux的基本命令入门
## 目录切换命令
### 目录结构
- / (根目录)
  - bin（binaries）存放二进制可执行文件
  - boot 存放用于系统引导时使用的各种文件
  - dev （devices）用于存放设备文件
  - etc （etcetera）存放系统配置文件
  - home 存放用户文件的根目录
    - 每个用户的根目录的存放的位置，home下创建每个用户的根目录。
    - 例如：用户是test，那么在home下就会存在一个叫test的目录
  - lib （library）存放跟文件系统中的程序运行所需要的共享库及内核模块
  - sbin （super user binaries）存放二进制可执行文件，只有root才能访问
  - usr （unix shared resources）用于存放共享的系统资源
  - var （variable）用于存放运行时需要改变数据的文件
  - ...
### 目录切换命令
cd usr		切换到该目录下usr目录
cd ../		切换到上一层目录
cd /		切换到系统根目录
cd ~		切换到用户主目录
cd -		切换到上一个所在目录
<!--more-->
## 目录的操作命令
### 增加目录操作
命令：mkdir 目录名称
示例：在根目录 / 下 mkdir test，就会在根目录 /下产生一个test问目录
![Alt](https://github.com/little-eight-china/image/blob/master/bdbk/linux/add.png?raw=true)
### 查看目录
命令：ls [-al] 父目录
示例：在根目录 / 下使用ls，可以看到该目录下的所有的目录和文件
![Alt](https://github.com/little-eight-china/image/blob/master/bdbk/linux/query.png?raw=true)
示例：在根目录 / 下使用ls -a，可以看到该目录下的所有文件和目录，包括隐藏的
![Alt](https://github.com/little-eight-china/image/blob/master/bdbk/linux/q21.png?raw=true)
示例：在根目录 / 下使用ls -l（ls -l 可以缩写成ll），可以看到该目录下的所有目录和文件的详细信息
![Alt](https://github.com/little-eight-china/image/blob/master/bdbk/linux/query2.png?raw=true)
### 寻找目录
命令：find 目录 参数
示例：查找/root下的与test相关的目录(文件)  find /root -name ‘test*’
### 修改目录的名称
命令：mv 目录名称 新目录名称（注意：mv的语法不仅可以对目录进行重命名而且也可以对各种文件，压缩包等进行	重命名的操作）
示例：test目录下有一个oldTest目录，使用mv oldTest newTest命令修改
![Alt](https://github.com/little-eight-china/image/blob/master/bdbk/linux/mv.png?raw=true)
### 移动目录的位置---剪切
命令：mv 目录名称 目录的新位置（注意：mv语法不仅可以对目录进行剪切操作，对文件和压缩包等都可执行剪切操作）
示例：在test下将newTest目录剪切到 /usr下面，使用mv newTest /usr
![Alt](https://github.com/little-eight-china/image/blob/master/bdbk/linux/move.png?raw=true)

### 拷贝目录
命令：cp -r 目录名称 目录拷贝的目标位置 -----r代表递归拷贝（注意：cp命令不仅可以拷贝目录还可以拷贝文件，压缩包等，拷贝文件和压缩包时不	用写-r递归）
示例：将/usr下的newTest拷贝到根目录下的test中，使用cp -r /usr/newTest /test
![Alt](https://github.com/little-eight-china/image/blob/master/bdbk/linux/copy.png?raw=true)

### 删除目录
命令：rm [-rf] 目录
示例：删除/usr下的newTest，进入/usr下使用rm -r newTest
![Alt](https://github.com/little-eight-china/image/blob/master/bdbk/linux/delete1.png?raw=true)
示例：删除/test下的newTest而不需要询问强制删除，在/test下使用rm -rf newTest
![Alt](https://github.com/little-eight-china/image/blob/master/bdbk/linux/delete2.png?raw=true)
注意：rm不仅可以删除目录，也可以删除其他文件或压缩包，为了增强大家的记忆，无论删除任何目录或文件，都直接使用rm -rf 目录/文件/压缩包

## 文件的操作命令
### 文件的创建
命令：touch 文件名称  ----- 空文件
示例：在test目录下创建一个空文件 touch aaa.txt
![Alt](https://github.com/little-eight-china/image/blob/master/bdbk/linux/add2.png?raw=true)
### 文件的查看
命令：cat/more/less/tail 文件
示例：使用cat查看/etc/sudo.conf文件，只能显示最后一屏内容
![Alt](https://github.com/little-eight-china/image/blob/master/bdbk/linux/query3.png?raw=true)
示例：使用more查看/etc/sudo.conf文件，可以显示百分比，回车可以向下一行，	空格可以向下一页，q可以退出查看
![Alt](https://github.com/little-eight-china/image/blob/master/bdbk/linux/query4.png?raw=true)
示例：使用less查看/etc/sudo.conf文件，可以使用键盘上的PgUp和PgDn向上	和向下翻页，q结束查看
![Alt](https://github.com/little-eight-china/image/blob/master/bdbk/linux/query5.png?raw=true)
示例：使用tail -10 查看/etc/sudo.conf文件的后10行，Ctrl+C结束
![Alt](https://github.com/little-eight-china/image/blob/master/bdbk/linux/query6.png?raw=true)
注意：命令 tail -f 文件 可以对某个文件进行动态监控，例如tomcat的日志文件，	会随着程序的运行，日志会变化，可以使用tail -f catalina-2018-11-11.log 监控	文件的变化 

### 修改文件的内容
命令：vim 文件
示例：编辑/test下的aaa.txt文件，使用vim aaa.txt
进入文件内容后，此时并不能编辑，因为此时处于命令模式，点击键盘i/a/o进入编辑模式，可以	编辑文件。编辑完成后，按下Esc，退回命令模式，此时文件虽然已经编辑完成，但是没有保存，需输入冒号：进入底行模式，在底行模	式下输入wq代表写入内容并退出，即保存；输入q!代表强制退出不保存。
关于vim使用过程：
在实际开发中，使用vim编辑器主要作用就是修改配置文件
vim 文件------>进入文件----->命令模式------>按i进入编辑模式----->编辑文件	------->按Esc进入底行模式----->输入:wq/q!
（关于vim的操作，这里只做最简单的阐述）

## 压缩文件的操作命令
### 打包并压缩文件
Linux中的打包文件一般是以.tar结尾的，压缩的命令一般是以.gz结尾的。
而一般情况下打包和压缩是一起进行的，打包并压缩后的文件的后缀名一般.tar.gz。
命令：tar -zcvf 打包压缩后的文件名 要打包压缩的文件
其中：z：调用gzip压缩命令进行压缩
  c：打包文件
  v：显示运行过程
  f：指定文件名
示例：打包并压缩/test下的所有文件 压缩后的压缩包指定名称为xxx.tar.gz
tar -zcvf xxx.tar.gz aaa.txt bbb.txt ccc.txt
或：tar -zcvf xxx.tar.gz /test/*
![Alt](https://github.com/little-eight-china/image/blob/master/bdbk/linux/dabao1.png?raw=true)
### 解压压缩包
命令：tar [-xvf] 压缩文件
其中：x：代表解压
示例：将/test下的xxx.tar.gz解压到当前目录下
tar -xvf xxx.tar.gz
![Alt](https://github.com/little-eight-china/image/blob/master/bdbk/linux/jieya1.png?raw=true)
示例：将/test下的xxx.tar.gz解压到根目录/usr下
tar -xvf xxx.tar.gz -C /usr------C代表指定解压的位置
![Alt](https://github.com/little-eight-china/image/blob/master/bdbk/linux/jieya2.png?raw=true)
## 其他命令
### 显示当前所在位置
命令：pwd
![Alt]()
### 搜索命令
命令：grep 要搜索的字符串 要搜索的文件
示例：搜索/usr/sudu.conf文件中包含字符串to的行
![Alt](https://github.com/little-eight-china/image/blob/master/bdbk/linux/sousuo1.png?raw=true)
示例：搜索/usr/sudu.conf文件中包含字符串to的行 to要高亮显示
![Alt](https://github.com/little-eight-china/image/blob/master/bdbk/linux/sousuo2.png?raw=true)
### 管道命令
命令：|   将前一个命令的输出作为本次目录的输入
示例：查看当前系统中所有的进程中包括system字符串的进程
![Alt](https://github.com/little-eight-china/image/blob/master/bdbk/linux/sousuo.png?raw=true)

### 查看进程
命令：ps -ef
示例：查看当前系统中运行的进程 

### 杀死进程
命令：kill -9 进程的pid

### 网络通信命令
查看当前系统的网卡信息：ifconfig
![Alt](https://github.com/little-eight-china/image/blob/master/bdbk/linux/ifconfig.png?raw=true)

查看与某台机器的连接情况：ping
![Alt](https://github.com/little-eight-china/image/blob/master/bdbk/linux/ping.png?raw=true)
查看当前系统的端口使用：netstat -an
![Alt](https://github.com/little-eight-china/image/blob/master/bdbk/linux/nes.png?raw=true)

# Linux的权限命令
## 权限说明
权限是Linux中的重要概念，每个文件/目录等都具有权限，通过ls -l命令我们可以	查看某个目录下的文件或目录的权限
示例：在随意某个目录下ls -l（用缩写。。这里为了美观）
![Alt](https://github.com/little-eight-china/image/blob/master/bdbk/linux/quanxian0.png?raw=true)
第一列的内容的信息解释如下：
![Alt](https://github.com/little-eight-china/image/blob/master/bdbk/linux/quanxian.png?raw=true)
- 文件的类型:
  -  d：代表目录
  - -：代表文件
  - l：代表链接（可以认为是window中的快捷方式）
  - l：代表链接（可以认为是window中的快捷方式）
后面的9位分为3组，每3位置一组，分别代表属主的权限，与当前用户同组的	用户的权限，其他用户的权限
  - r：代表权限是可读，r也可以用数字4表示
  - w：代表权限是可写，w也可以用数字2表示
  - x：代表权限是可执行，x也可以用数字1表示
![Alt](https://github.com/little-eight-china/image/blob/master/bdbk/linux/quanxian2.jpg?raw=true)
## 修改文件/目录的权限的命令：chmod
示例：修改/test下的aaa.txt的权限为属主有全部权限，属主所在的组有读写权限，
其他用户只有读的权限
chmod u=rwx,g=rw,o=r aaa.txt
![Alt](https://github.com/little-eight-china/image/blob/master/bdbk/linux/chmod.png?raw=true)
上述示例还可以使用数字表示：
chmod 764 aaa.txt

## 切换用户
有时候用户并不是root用户，没有权限修改文件权限
命令：sudo su ，然后输入密码，回车
root用户转回普通用户
命令：exit
