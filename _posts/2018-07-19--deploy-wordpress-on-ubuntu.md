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
```

拷贝wordpress的代码过去（我干嘛不直接解压到这里呢，真是傻。。。）：

```
$ cp -r ~/wordpress/* /data/wwwroot/default/
```

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

<br>
完


## 0x04 参考链接
- [主要参考文档](https://oneinstack.com/docs/lnmpstack-image-guide/)
- [有点旧但是还是相当有用的文档](https://codex.wordpress.org/zh-cn:%E5%AE%89%E8%A3%85_WordPress)
- [phpMyAdmin使用教程](https://blog.csdn.net/u012767761/article/details/78238487)