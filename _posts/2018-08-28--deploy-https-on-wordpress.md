---
layout:     post
title:      "WordPress全站部署HTTPS"
date:       2018-08-28
author:     "winson"
tags:
    - wordpress,nginx
---

## WordPress全站部署HTTPS
由于做这件事的时候遇到了一些坑，特此记录。

## 0x00 部署环境
我使用的是[oneinstack](https://oneinstack.com/)一键安装包，Linux + Nginx + Apache + MySQL + PHP， nginx处理静态资源，apache处理PHP。<br>
Wordpress如果部署在反向代理后会有一些问题，要对wordpress代码做一些特殊处理，下面会说到。<br>
另外Chrome的缓存会影响到结果判断，但又不方便全局清理(我懒，不想去查)，于是使用了Microsoft Edge，缓存随便清。

## 0x01 准备证书。
两种方法，要么自己生成CSR然后提交给签发机构获取CRT和KEY，要么你去买SSL证书的时候签发机构直接就是给你CRT和KEY了。我就是后面那种情况。
<br>
如果给你的CRT和KEY是字符串形式的，自己下回来，第一个保存成.crt后缀名，第二个保存成.key后缀名。注意**第一行**与**最后一行**例如`-----BEGIN CERTIFICATE-----`或者`-----BEGIN PUBLIC KEY-----`这里要加个换行，否则后面nginx加载的时候会报证书格式错误。

## 0x02 证书放哪里的问题
我放了/etc/ssl/private，但有个问题，由于nginx运行的时候要访问这个两个证书，而nginx运行的用户是www，所以必须保证www（nginx worker运行的用户）有权限去读这两个文件。
<br>
于是新加一个用户组`ssl-cert`，专门用来访问这些证书文件：

```
$ sudo groupadd ssl-cert
$ sudo usermod -a -G ssl-cert www
$ sudo usermod -a -G ssl-cert root
$ sudo chown root:ssl-cert /etc/ssl/private/mysite.crt
$ sudo chown root:ssl-cert /etc/ssl/private/mysite.key
$ sudo chmod 740 /etc/ssl/private/mysite.crt
$ sudo chmod 740 /etc/ssl/private/mysite.key
```

不知道为什么这时候如果用`nginx -t`去测试的话还是会报错(Permission Denied)，但是如果`sudo service nginx reload/restart`就没问题(如果有问题这命令会提示失败)。可能前者还是用当前用户吧(当前用户既不是root也不是www)。

## 0x03 配置nginx
在配置之前我们先来回顾一下我们的网站架构：Linux + Nginx + Apache + MySQL + PHP，请求会先到nginx，然后到apache，然后到php。由于我们是强制https，所以一定有http重定向到https的配置。这里有两个选择：<br>
1. nginx直接重定向
2. apache用.htaccess重定向

为简单起见，我把所有杂七杂八的东西都放nginx处理了，后端只负责具体的请求处理。<br>
整理一下nginx需要做的事情：
1. 启用ssl
2. 重定向http到https
3. 重定向mysite.com到www.mysite.com
4. 分离静态资源的访问与动态资源的访问

先放最终的配置(期间试了好多种不同的写法，我觉得这种http与https分离的写法比较清晰)，首先是nginx.conf

```
user www www;
worker_processes auto;

error_log /data/wwwlogs/error_nginx.log crit;
pid /var/run/nginx.pid;
worker_rlimit_nofile 51200;

events {
  use epoll;
  worker_connections 51200;
  multi_accept on;
}

http {
  include mime.types;
  default_type application/octet-stream;
  server_names_hash_bucket_size 128;
  client_header_buffer_size 32k;
  large_client_header_buffers 4 32k;
  client_max_body_size 1024m;
  client_body_buffer_size 10m;
  sendfile on;
  tcp_nopush on;
  keepalive_timeout 120;
  server_tokens off;
  tcp_nodelay on;

  fastcgi_connect_timeout 300;
  fastcgi_send_timeout 300;
  fastcgi_read_timeout 300;
  fastcgi_buffer_size 64k;
  fastcgi_buffers 4 64k;
  fastcgi_busy_buffers_size 128k;
  fastcgi_temp_file_write_size 128k;
  fastcgi_intercept_errors on;

  #Gzip Compression
  gzip on;
  gzip_buffers 16 8k;
  gzip_comp_level 6;
  gzip_http_version 1.1;
  gzip_min_length 256;
  gzip_proxied any;
  gzip_vary on;
  gzip_types
    text/xml application/xml application/atom+xml application/rss+xml application/xhtml+xml image/svg+xml
    text/javascript application/javascript application/x-javascript
    text/x-json application/json application/x-web-app-manifest+json
    text/css text/plain text/x-component
    font/opentype application/x-font-ttf application/vnd.ms-fontobject
    image/x-icon;
  gzip_disable "MSIE [1-6]\.(?!.*SV1)";

  #If you have a lot of static files to serve through Nginx then caching of the files' metadata (not the actual files' contents) can save some latency.
  open_file_cache max=1000 inactive=20s;
  open_file_cache_valid 30s;
  open_file_cache_min_uses 2;
  open_file_cache_errors on;

######################## default ############################
#  server {
#    listen 80;
#    server_name _;
#    access_log /data/wwwlogs/access_nginx.log combined;
#    root /data/wwwroot/default;
#    index index.html index.htm index.php;
#    #error_page 404 /404.html;
#    #error_page 502 /502.html;
#    location /nginx_status {
#      stub_status on;
#      access_log off;
#      allow 127.0.0.1;
#      deny all;
#    }
#    location / {
#      try_files $uri @apache;
#    }
#    location @apache {
#      proxy_pass http://127.0.0.1:88;
#      include proxy.conf;
#    }
#    location ~ [^/]\.php(/|$) {
#      proxy_pass http://127.0.0.1:88;
#      include proxy.conf;
#    }
#    location ~ .*\.(gif|jpg|jpeg|png|bmp|swf|flv|mp4|ico)$ {
#      expires 30d;
#      access_log off;
#    }
#    location ~ .*\.(js|css)?$ {
#      expires 7d;
#      access_log off;
#    }
#    location ~ /\.ht {
#      deny all;
#    }
#  }
#
########################### vhost #############################
  include vhost/*.conf;
}
```

其实就是注释掉了原来的default的内容。<br>

然后是www.mysite.com.conf
```
server {
  listen 80;
  server_name mysite.com www.mysite.com;
  return 301 https://www.mysite.com$request_uri;
}

server {
  listen 443 ssl;
  server_name mysite.com www.mysite.com;

  ssl on;
  ssl_certificate /etc/ssl/private/mysite.crt;
  ssl_certificate_key /etc/ssl/private/mysite.key;
  ssl_session_timeout  5m;
  ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
  ssl_ciphers  ECDHE-RSA-AES128-GCM-SHA256:HIGH:!aNULL:!MD5:!RC4:!DHE;
  ssl_prefer_server_ciphers  on;

  access_log /data/wwwlogs/www.mysite.com_nginx.log combined;
  index index.html index.htm index.php;
  root /data/wwwroot/www.mysite.com;

  location / {
    try_files $uri @apache;
  }
  location @apache {
    proxy_pass http://127.0.0.1:88;
    include proxy.conf;
  }
  location ~ .*\.(php|php5|cgi|pl)?$ {
    proxy_pass http://127.0.0.1:88;
    include proxy.conf;
  }
  location ~ .*\.(gif|jpg|jpeg|png|bmp|swf|flv|mp4|ico)$ {
    expires 30d;
    access_log off;
  }
  location ~ .*\.(js|css)?$ {
    expires 7d;
    access_log off;
  }
  location ~ /\.ht {
    deny all;
  }
}
```

改完后重新加载`sudo service nginx reload`，或者保险一点`sudo service nginx restart`
这样改完的话会可能出现几个现象：
1. Chrome提示该网站并非完全安全
2. 无法进入wp-admin页面
3. F12查看具体信息，Console页面会提示Mix Content错误，Network页面会看到我很多红色的加载不出来的资源

这其实是因为wordpress页面资源的加载很多都是写死http，要实现全站https，还要看看怎么处理workpress。

## 0x04 配置wordpress
直接Google搜索workpress怎么针对https做改动，貌似就是在后台设置一下网站路径，
![](https://winson08Li.github.io/img/in-post/2018-08-28--deploy-https-on-wordpress/WordPress-General-Settings.png)

改完Microsoft Edge清理缓存刷新页面，这时候发现页面进不去了！！提示重定向太多，切回chrome一看果然全是302重定向，无限循环。<br>

Google之，发现了[这个答案](https://stackoverflow.com/questions/27193575/wordpress-cloudfront-flexible-ssl-ends-up-in-redirect-loop-https/27193576#27193576)，意思就是wordpress不知道前面有反向代理已经帮他处理了所有的需要重定向的事情，因为wordpress前面还有个apache，nginx传给apache的是http，然后你一改wordpress的链接，wordpress就把http重定向成https了，然后就引发了无限循环。
<br>
解决方法是改wp-config.php<br>
在`require_once(ABSPATH . 'wp-settings.php');`这一行之前加上下面这行：

```
if ($_SERVER['HTTP_X_FORWARDED_PROTO'] == 'https') $_SERVER['HTTPS']='on';
```

这样就能解决问题。

Edge清理缓存刷新，发现可以进去页面了，证明刚才的修改起了作用。<br>
然而用Chrome看了一下仍然提示“该网站并非完全安全”，右键查看源代码发现依然有一些链接是使用的http，想到了wordpress用的插件可能也会有影响，于是Google之，发现确实是。

## 0x05 配置wordpress其他的插件
由于我用了不少wordpress的插件，有些插件的链接需要处理一下。幸好我使用的Elementor官方有处理的办法。
Elementor在Tools菜单里提供了一个工具，该工具可以进行全局替换字符串，具体操作参考[该链接](https://docs.elementor.com/article/218-i-changed-the-url-of-my-website-and-elementor-does-not-work-anymore).
![](https://winson08Li.github.io/img/in-post/2018-08-28--deploy-https-on-wordpress/elementor-replace-url.png)

完了后Chrome打开，提示网站安全~O(∩_∩)O~~~~~~~~~~~~

## 0x06 题外话
其实还有一种更简便的方式，但是我没试过，是我弄完了整理资料的时候才注意到的。<br>
Oneinstack这个工具包支持创建https的网站，但是是用的自签名证书，我一度以为这样不适合我的需求。但其实只要把自己买的证书替换掉自签名的crt跟key就好了，省去自己配置nginx以及证书文件怎么处理的事情，当然文件名字要改一下。<br>
但是wordpress那里还是不能省的。

全文完。


## 0x06 参考链接
- [nginx documentation](http://nginx.org/en/docs/)
- [How nginx processes a request](http://nginx.org/en/docs/http/request_processing.html)
- [全站 HTTPS 没你想象的那么简单](http://www.cnblogs.com/mafly/p/allhttps.html)
- [Let's Encrypt 使用教程，免费的SSL证书，让你的网站拥抱 HTTPS](http://diamondfsd.com/lets-encrytp-hand-https/)
- [启用 https 简单免费的 Let's Encrypt SSL证书配置](https://segmentfault.com/a/1190000012343679)
- [I Changed the URL of My Website and Elementor Does Not Work Anymore](https://docs.elementor.com/article/218-i-changed-the-url-of-my-website-and-elementor-does-not-work-anymore)
- [WordPress + CloudFront Flexible SSL ends up in redirect loop (https)](https://stackoverflow.com/questions/27193575/wordpress-cloudfront-flexible-ssl-ends-up-in-redirect-loop-https/27193576#27193576)
- [How to create users and groups in Linux from the command line](https://www.techrepublic.com/article/how-to-create-users-and-groups-in-linux-from-the-command-line/)
- [Let's Encrypt 免费SSL证书教程](https://oneinstack.com/faq/letsencrypt/)