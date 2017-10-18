---
layout:     post
title:      "在Ubuntu16.04上编译OpenCV4Android"
date:       2017-10-12
author:     "winson"
tags:
    - Opencv
    - Android
---

# 在Ubuntu16.04上编译OpenCV4Android

## 0x00 准备源码
直接到github上下载最新版源码

``` shell
git clone https://github.com/opencv/opencv.git
```

## 0x01 编译环境准备
安装cmake

``` shell
$ sudo apt-get install cmake
```

但是有时候发现这样安装的不是最新版的cmake，opencv3.3的编译貌似需要的cmake版本比ubuntu官方仓库的要新。 直接cmake官网下载最新版的cmake然后configure然后make & make install

安装ant

``` shell
$ sudo apt-get install ant
```

autotools系列工具，这应该大家都有的。

环境变量：ANDROID_NDK， ANDROID_SDK， JAVA_HOME

## 0x02 编译

``` shell
$ cd cd <your-path-to-opencv-source>/platforms/script
$ ./cmake_android_arm.sh
```

> 这样默认编出来的是armv7a的版本。

## 0x03 使用opencv_contrib
代码下载:

``` shell
git clone https://github.com/opencv/opencv_contrib.git
```

然后按照上面的步骤正常编译，发现会有错误，大概是说有文件没找到
```
opencv_contrib/modules/xfeatures2d/src/boostdesc.cpp:646:37: fatal error: boostdesc_bgm.i: No such file or directory
```
Google一番发现原来是因为opencv的cmake编译脚本执行过程中会去用curl下载点东西，下载记录在build根目录(即构建目录)的CMakeDownloadLog.txt文件里。打开这个文件会发现用curl去下载一些文件的时候失败了。又Google一番发现是cmake用的curl没有ssl支持访问不了https。于是准备重新安装curl。

首先卸载系统的curl

``` shell
sudo apt-get remove curl
```

github下载curl源码

``` shell
https://github.com/curl/curl.git
```

然后开始编译curl

``` shell
./buildconf
./configure --with-ssl
make
sudo make install
```

这里有个坑，这样直接弄的话会出现curl不能用的情况，原因是curl跟其所使用的libcurl版本不匹配导致。可以用如下命令证实：

``` shell
curl --version
```

这样会输出curl以及使用的libcurl使用的版本号，类似于这样(这个是我找网上的，因为我忘记截图了...)：

``` shell
$ curl --version
curl 7.41.0 (x86_64-unknown-linux-gnu) libcurl/7.22.0 OpenSSL/1.0.1 zlib/1.2.3.4 libidn/1.23 librtmp/2.3
```

然后我们看看我们在用哪个curl:

``` shell
$ which curl
/usr/local/bin/curl
```

然后看看这个`curl`链接的是哪个`libcurl.so`:

``` shell
$ ldd /usr/local/bin/curl
```

发现其使用的并不是位于/usr/local/lib里面的(curl源码编译的安装位置)，用的是另外一个，估计是系统自带的。
于是又Google一番，找到解决办法：让系统先找/usr/local/lib里面的库。修改/etc/ld.so.conf这个文件，在`include /etc/ld.so.conf.d/*.conf`上面`增加一行include /usr/local/lib`

然后执行命令:

``` shell
$ ldconfig -v
```

再次运行`curl --version`，发现版本一致了。

于是继续opencv的编译。
仍然有错误，似乎刚才的修改没有起作用，但是我们用命令行测试curl确实是成功的。于是继续Google，发现是cmake的坑。

cmake内部有一个curl，默认情况下其使用的就是这个，如果你不改配置的话。

于是重新编译cmake：

``` shell 
$ ./configure --system-curl
make
sudo make install
```

然后重新再编译opencv。终于通过了这次！全部编译成功。

## Reference:

[Building OpenCV4Android from trunk](http://code.opencv.org/projects/opencv/wiki/Building_OpenCV4Android_from_trunk)

[Automatic hunter download fails with "Protocol "https" not supported or disabled in libcurl"](https://github.com/ruslo/hunter/issues/328)

[Libcurl not updated](https://stackoverflow.com/questions/36866583/libcurl-not-updated)

[curl: (48) An unknown option was passed in to libcurl](https://stackoverflow.com/questions/11678085/curl-48-an-unknown-option-was-passed-in-to-libcurl)