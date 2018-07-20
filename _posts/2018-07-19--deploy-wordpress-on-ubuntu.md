---
layout:     post
title:      "Ubuntu 16.04 部署 WordPress"
date:       2018-07-19
author:     "winson"
tags:
    - wordpress
---

## Ubuntu 16.04 部署 WordPress
由于公司的需要，需要一个wordpress站来做测试，特此记录。

## 0x00 买主机
直接阿里云撸了一台一个月80不到的云主机来做测试。注意买主机的时候可以到镜像市场选系统，我选了一个LNMP环境的镜像，地址在[这里](https://market.aliyun.com/products/53398003/cmjj025458.html?spm=5176.204674.1085795.1.yAMG6r)
还有完善的[文档](https://oneinstack.com/docs/lnmpstack-image-guide/)
<br>
可怕的大佬。。。
<br>
## 0x01 准备网站源码
主机搞定后直接ssh登陆，下载最新的wordpress代码：


```
$ wget http://wordpress.org/latest.tar.gz
```

然后解压`tar -xzvf latest.tar.gz`
<br>
备份原来的默认站到~/default_backup

```
$ mkdir ~/default_backup
$ cp -r /data/wwwroot/default/* ~/default_backup
$ rm -rf /data/wwwrott/default/*
```

拷贝wordpress的代码过去（我干嘛不直接解压到这里呢，真是傻。。。）：

```
$ cp -r ~/wordpress/* /data/wwwroot/default/
```

然后请**务必**更改目录的权限。我遇到下载wordpress插件的时候出现要求输入ftp用户名密码的问题，就是因为这个引起的。总之如果权限不对会有很多奇怪的问题，所以这步绝对绝对不能省。

```
$ chown -R www.www /data/wwwroot/
$ find /data/wwwroot/ -type d -exec chmod 755 {} \;
$ find /data/wwwroot/ -type f -exec chmod 644 {} \;
```

这样就OK拉！

## 0x02 准备数据库
直接用phpMyAdmin，按照文档输入`xxx.xx.xxx.xxx/phpMyAdmin/`出现404错误
原来是我刚才把默认站挪走了，当然找不到了，于是在`default`建了个新文件夹`nv`然后把原来默认站的内容copy过去了，继续：<br>
`xxx.xx.xxx.xxx/nv/phpMyAdmin/`<br>
出现登陆界面，输入默认数据库用户名密码，成功登陆。<br>

然后新建一个数据库，<br>
点击上面菜单`数据库`，输入`wordpress`，`排序规则`这里设成`utf8-general-ci`，其他不要动，然后点创建。

 > 从网上看了篇文字说要改编码不然中文乱码。谁知道公司需不需要中文呢...那就改吧，就是`排序规则`这里设成`utf8-general-ci`


然后创建数据库用户:<br>
点最上面的`服务器：localhost`，然后点`用户`，选`添加用户`，用户名(使用文本域)，host改成localhost，%表示任意主机，也没问题，不过这里我改成localhost了。密码(使用文本域)。
然后下面的权限都不选，点右下角执行。<br>

然后带上面`服务器:localhost`返回，选`数据库`，选中刚才新建的那个`检查权限`，看到我们新建的用户，然后点`编辑权限`，全选，执行（新建用户这里主要是参考了第二个文档）。

到这里就搞定了数据库的准备。

## 0x03 最后的配置
然后就是直接访问网站的IP地址`http://xxx.xx.xxx.xxx`，出现wordpress配置界面，随便配，完成后就可以了

## 0x04 额外的话
第一个是在配置nginx的虚拟主机上花了很多时间。一开始的时候不想改默认站，然后就打算新建一个虚拟服务器，操作到一半突然发现没有域名，想到nginx应该不至于只能用域名配置虚拟服务器，于是google了一番发现确实可以用其他方式（分别是基于端口与基于IP），然后搞了很久发现还是不行，防火墙规则端口过滤什么的都搞了，本地`curl http://localhost:8080`就可以，但是localhost换成外部IP/局域网IP就不行。无语了于是一怒之下替换了默认站。<br>
另外一个是自己傻逼，连接数据库不成功，原来自己密码填错了。。。<br>
PS：这个环境包真的好用，哈哈！

# 0x05 补充PureFtp的事项
刚说完这个环境包好用就给我捅娄子了。。。这个环境包里面的PureFtp有问题，会出现`unable to open the passwd file no such file or directory`的错误，重启pureftp无法解决问题，Google了很久以及自己试出了如下解决办法：

```
$ touch /usr/local/pureftpd/etc/pureftpd.passwd
$ rm -rf /usr/local/pureftpd/etc/pureftpd_psss.tmp
$ /usr/local/pureftpd/bin/pure-pw mkdb -F /usr/local/pureftpd/etc/pureftpd.pdb
```

这样就好啦！然后直接用环境包提供的脚本就能新建用户！


全文完。


## 0x06 参考链接
- [主要参考文档](https://oneinstack.com/docs/lnmpstack-image-guide/)
- [有点旧但是还是相当有用的文档](https://codex.wordpress.org/zh-cn:%E5%AE%89%E8%A3%85_WordPress)
- [phpMyAdmin使用教程](https://blog.csdn.net/u012767761/article/details/78238487)
- [pureftp的问题参考连接](https://superuser.com/questions/1080220/pure-pw-error-unable-to-open-the-passwd-file-no-such-file-or-directory)