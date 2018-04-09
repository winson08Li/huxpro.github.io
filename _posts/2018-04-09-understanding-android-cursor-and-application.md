---
layout:     post
title:      "理解Android中的Cursor及其应用"
date:       2018-04-09
author:     "winson"
tags:
    - Android
---

近来在做项目的时候发现在APP各组件间共享数据是件麻烦事，尤其涉及到异步更新的时候，就会出现各种烦人的listener，各种callback。于是想想是不是有什么统一接口来实现自动更新之类的。于是就想到了Cursor，于是就想到了用ContentProvider来实现这个功能。
<!-- TOC -->

- [0x00 从如何自定义一个ContentProvider开始](#0x00-从如何自定义一个contentprovider开始)
- [0x01 无法避免的Uri](#0x01-无法避免的uri)
    - [Uri的组成](#uri的组成)
    - [Uri的匹配与UriMatcher](#uri的匹配与urimatcher)
- [0x02 Cursor的动态更新](#0x02-cursor的动态更新)
    - [原理](#原理)
    - [我们常用的API LoaderManager](#我们常用的api-loadermanager)

<!-- /TOC -->
## 0x00 从如何自定义一个ContentProvider开始
就像网上很多教程

## 0x01 无法避免的Uri
### Uri的组成
一旦使用上Cursor，就无法避免要使用这个类，然后就无法避免要搞清楚Android是如何组织Uri的。
[这篇文章](https://blog.csdn.net/harvic880925/article/details/44679239)说得比较清楚了，就不再重复了，这里只说一下重点。

基本结构是这样的

```
[scheme:][//authority][path][?query][#fragment]
```

其中authority又能分成`[userinfo@]host:port`的形式，所以其结构也可以是这样的
```
[scheme:][//[userinfo@]host:port][path][?query][#fragment]
```
userinfo一般写成`name:pwd`的形式。

其各部分的意义基本跟平时见到的网络uri差不多，就后面那个`[#fragment]`不一样。用`#`分割的后面那部分都是属于`fragment`。
举个栗子：
```
http://www.wtf.com:8080/yourpath/fileName.htm?index=10&path=32&id=4#winson
```

`http`就是`scheme`, `www.wtf.com:8080`就是authority，`/yourpath/fileName.htm`这部分就是`path`, `query`是一个键值对，多个键值对间用`&`分隔。
`winson`就是`fragment`。
<br>
都很简单直接。
<br>
那在Android中怎么操作这种Uri呢？Android的Uri类提供了众多操作Uri的方法，仅列举其中一些常用的：

- getScheme
  返回其中的authority部分，在上例中即`http`
- getAuthority
  返回其中的authority部分，在上例中就是`www.wtf.com:8080`
- getHost/getPort
  返回主机以及端口号，就是`www.wtf.com`以及`8080`
- getPath
  返回`/yourpath/fileName.htm`这部分
- getQuery 
  返回`index=10&path=32&id=4`这部分
- getFragment
  返回`winson`
- getPathSegments
  返回Path的所有分段，上例中就是返回`yourpath`, `fileName.htm`
- getLastPathSegment
  这个就跟上面那个一样，只是返回的是path中最后的那个元素，即`fileName.htm`
- getQueryParameterNames/getQueryParameters/getQueryParameter/getBooleanQueryParameter
  这几个都是类似的，只是方便我们取出URI中的查询参数
- withAppendedPath
  拼接一个`path`到uri上。很好理解，比如有一个uri是`content://my.app.com/media`，我想在后面拼个ID 20变成这样`content://my.app.com/media/20`，直接这么用`Uri.withAppendedPath(uri, "20");`可以简单理解为字符串拼接。
  其实还可以用另一个工具类`ContentUris`，`ContentUris.withAppendedId(uri, 20)`，一样的。

### Uri的匹配与UriMatcher
首先是匹配规则，可以参看[Android官网的说明](https://developer.android.com/guide/topics/providers/content-provider-creating.html?hl=zh-cn)。
简单说来就是记住两点就可以了：
- 没有特殊符号的URI直接全部匹配
- 符号`*`匹配由任意长度的任何有效字符组成的字符串
- 符号`#`匹配由任意长度的数字字符组成的字符串

`#`通常用于匹配单行内容，即某个ID的记录
<br>
`UriMatcher`就是基于以上规则而来，用法就是：
- 1. 往里面加规则
- 2. 调用match(uri)返回一个匹配码，告诉使用者匹配了哪种模式

所以我们一般的用法就是：<br>
第一，加规则：<br>
```

private static final int MATCH_TABLE3 = 1;
private static final int MATCH_TABLE3_ID = 2;
private static final int MATCH_TABLE3_ATTR1 = 3;

static {
    sUriMatcher.addURI("com.example.app.provider", "table3", MATCH_TABLE3);
    sUriMatcher.addURI("com.example.app.provider", "table3/#", MATCH_TABLE3_ID);
    sUriMatcher.addURI("com.example.app.provider", "table3/#/attr1", MATCH_TABLE3_ATTR1);
}
```
第二，用：<br>
```
public Cursor query(@NonNull Uri uri, @Nullable String[] projection, @Nullable String selection, @Nullable String[] selectionArgs, @Nullable String sortOrder) {
    ...
    ...
    switch (sUriMatcher.match(uri)) {
        case MATCH_TABLE3:
            ...
            break;
        case MATCH_TABLE3_ID:
            ...
            break;
        case MATCH_TABLE3_ATTR1:
            ...
            break;
    }
    ...
    ...
}
```

我一般不用`*`，因为觉得没这个必要吧。

## 0x02 Cursor的动态更新
### 原理
### 我们常用的API LoaderManager