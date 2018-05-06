---
layout:     post
title:      "使用Nexus部署内部Maven私服 --- Ubuntu 16.04 安装Nexus"
date:       2018-05-06
author:     "winson"
tags:
    - Nexus
    - Maven
---

## 使用Nexus部署内部Maven私服 --- Ubuntu 16.04 安装Nexus
<!-- TOC -->

- [使用Nexus部署内部Maven私服 --- Ubuntu 16.04 安装Nexus](#使用nexus部署内部maven私服-----ubuntu-1604-安装nexus)
- [0x00 准备安装文件](#0x00-准备安装文件)
- [0x00 新建Nexus用户](#0x00-新建nexus用户)
- [0x01 修改Nexus与环境变量配置](#0x01-修改nexus与环境变量配置)
- [0x02 运行](#0x02-运行)
- [0x03 配置开机运行](#0x03-配置开机运行)
- [0x04 参考链接](#0x04-参考链接)

<!-- /TOC -->

## 0x00 准备安装文件
到[Nexus官方下载地址](https://www.sonatype.com/download-oss-sonatype)下载最新版Nexus，现在我下载的是`nexus-3.11.0-01 `这个版本。<br> 
到'/opt'新建目录'nexus'，然后把文件下载到那里：

```
cd /opt
sudo mkdir nexus
sudo wget https://sonatype-download.global.ssl.fastly.net/repository/repositoryManager/3/nexus-3.11.0-01-unix.tar.gz
tar -zxvf nexus-3.11.0-01-unix.tar.gz
```

解压后得到的文件结构是这样的：

```
~$ ls /opt/nexus/ -a
.  ..  nexus-3.11.0-01  nexus-3.11.0-01-unix.tar.gz  sonatype-work
```

压缩文件可以删了。

## 0x00 新建Nexus用户
新建一个专门用来运行nexus的用户:

```
~$ useradd nexus
~$ usermod -aG sudo nexus
~$ passwd nexus
```

这样新建出来的用户是三无用户：没有home，没有密码，没有系统shell。上面是给他加了密码。同时加入了sudo列表（其他文档安装nexus的时候有）。这里没有创建主目录，如果要的话像下面那样操作：

```
~$ sudo -i                            #to get root privileges
~$ mkdir /home/nexus                  #to create the directory /home/linda
~$ cp -rT /etc/skel /home/nexus         #to populate /home/linda with default files and folders
~$ chown -R nexus:nexus /home/nexus   #to change the owner of /home/linda to user linda
```

> 直接用adduser会比较好，毕竟新手

然后修改目录权限：

```
~$ sudo chown -R nexus:nexus /opt/nexus
```

## 0x01 修改Nexus与环境变量配置

```
~$ sudo vi /opt/nexus/nexus-3.11.0-01/bin/nexus
```

找到`run_as_user`这一行，改成`run_as_user='nexus'`。<br>
修改运行权限:

```
~$ chmod a+x /opt/nexus/nexus-3.11.0-01/bin/nexus
```

新增全局环境变量`NEXUS_HOME`：

```
~$ sudo vi /etc/profile
```

加入`NEXUS_HOME="/opt/nexus/nexus-3.11.0-01"`<br>
保存退出。<br>
马上生效`source /etc/profile`<br>

## 0x02 运行
直接执行就好了：

```
~$ cd /opt/nexus/nexus-3.11.0-01/bin
~$ ./nexus start
```

浏览器访问:`http://your-server-ip-addr:8081`

## 0x03 配置开机运行
按照官网文档准备用chkconfig配置开机运行，但是发现ubuntu自12.04之后不再支持chkconfig，但有个替代品叫`sysv-rc-conf`，作用几乎一样，但是在执行service nexus start命令的时候出现如下错误`Failed to start nexus.service: Unit nexus.service not found. `

当时我的操作是这样的：

```
~$ ln -s $NEXUS_HOME/bin/nexus /etc/init.d/nexus
~$ sysv-rc-conf nexus on
sudo service nexus start
```

出了上面说的错误提示，然后我怂了改成用update.rc了:

```
~$ cd /etc/init.d
~$ update-rc.d nexus defaults
~$ service nexus start
~$ service nexus status
```

然后我看到了输出中是这样的：
```
nexus.service - LSB: nexus
   Loaded: loaded (/etc/init.d/nexus; bad; vendor preset: enabled)
   Active: active (exited) since 日 2018-05-06 17:52:07 CST; 52min ago
     Docs: man:systemd-sysv-generator(8)
  Process: 734 ExecStart=/etc/init.d/nexus start (code=exited, status=0/SUCCESS)

```

exited ?!退出了？<br>
但其实是已经跑起来的，因为`nexus`这个程序貌似不是个守护进程?

然后我尝试重启，服务依然可以访问。

完。

## 0x04 参考链接
- [官网安装文档](https://books.sonatype.com/nexus-book/3.1/reference/install.html#service-linux)
- [sysv-rc-conf的介绍](https://blog.csdn.net/gatieme/article/details/45251389)
- [使用systemd的方式配置开机启动](https://ha.cker.ir/2017/10/13/installing-nexus-repository-manager-oss-3-ubuntu-16-04-nginx/)
- [另一个使用systemd配置Nexus的例子](https://blog.csdn.net/haiqinma/article/details/77507524)
- [有关service、update-rc.d、systemctl的关系与区别](https://blog.csdn.net/qq_37993487/article/details/79868857)
- [有关System V机制的系统启动项管理](https://zhuanlan.zhihu.com/p/32847078)
- 蛋疼的system v、upstart和systemd
    1. [浅析 Linux 初始化 init 系统](http://www.cnblogs.com/shanyou/p/4508190.html)
    2. [浅析 Linux 初始化 init 系统，第 1 部分 sysvinit](https://www.ibm.com/developerworks/cn/linux/1407_liuming_init1/index.html)
    3. [浅析 Linux 初始化 init 系统，第 2 部分 UpStart](https://www.ibm.com/developerworks/cn/linux/1407_liuming_init2/index.html)
    4. [浅析 Linux 初始化 init 系统，第 3 部分 Systemd](https://www.ibm.com/developerworks/cn/linux/1407_liuming_init3/index.html)
    5. [SystemV, SystemD和Upstart的简单比较](https://bitsflow.org/os/init/)