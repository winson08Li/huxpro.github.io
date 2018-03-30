---
layout:     post
title:      "Android使用cmake编译fftw-3.3.7"
date:       2018-03-30
author:     "winson"
tags:
    - fftw
    - Android
---

# Android使用cmake编译fftw-3.3.7

[上篇](https://winson08li.github.io/2017/10/09/cross-complie-fftw3.3.6-for-android/)博客介绍了怎么使用cmake在Android Studio里面编译ffttw3库，这次介绍怎么脱离Android Studio来编译。至于为什么选择3.3.7的，其实两版代码的编译可以认为没有区别，只是距离上次编译有一段时间了，这次又遇到需要fftw库的情况，就干脆用最新版的了。
<br><br>
同样从[官网](http://fftw.org/download.html)下载fftw-3.3.7的源代码。直接解压tar zxvf fftw-3.3.7.tar.gz
<br><br>
然后把进入解压后的目录，直接把上次修改后的CMakeLists.txt复制过来，然后在最前面加一句`cmake_minimum_required (VERSION 3.4.1)`
<br><br>
新建一个build目录，进入build目录，然后执行如下命令:
`cmake -DCMAKE_TOOLCHAIN_FILE=/path-to-your-ndk/build/cmake/android.toolchain.cmake ..`
回车，然后就会自动执行编译，完了后在build下生成libfftw3.so