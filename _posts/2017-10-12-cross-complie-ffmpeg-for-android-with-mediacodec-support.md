---
layout:     post
title:      "在Ubuntu16.04上编译ffmpeg"
date:       2017-10-12
author:     "winson"
tags:
    - ffmpeg
    - Android
---

# 在Ubuntu16.04上编译ffmpeg

## 0x00 准备源码
直接到github上下载最新版源码

``` shell
git clone https://github.com/opencv/opencv.git
```

## 0x01 编译环境准备
安装cmake
> sudo apt-get install cmake

但是有时候发现这样安装的不是最新版的cmake，直接cmake官网下载最新版的cmake然后configure然后make & make install


