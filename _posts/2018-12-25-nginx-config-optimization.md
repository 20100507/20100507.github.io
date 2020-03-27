---
layout: post
title: "Nginx配置优化"
date: 2018-12-25  
tag: Nginx
---

### 前言
    
	系统部署难免需要对nginx做一些配置优化,记录如下.

### 优化

**Request Entity Too Large Ingress Nginx**

* 普通nginx配置

```
  client_max_body_size 20M;
```
* ingress nginx配置

```
  apiVersion: extensions/v1beta1
  kind: Ingress
  metadata:
    name: ingress-nginx
    namespace: saas-pro
    annotations:
      kubernetes.io/ingress.class: "nginx"
      nginx.ingress.kubernetes.io/proxy-body-size: "50m"
```

**开启gzip压缩**

* 普通nginx配置

```
  gzip on;
  gzip_min_length 1k;
  gzip_comp_level 3;
  gzip_types text/plain application/javascript application/x-javascript text/css application/xml text/javascript application/x-httpd-php image/jpeg image/gif image/png application/vnd.ms-fontobject font/ttf font/opentype font/x-woff image/svg+xml;
  gzip_vary on;   
  gzip_buffers 32 4k;
  
```

**Get请求太长&Post Body太长**

* 普通nginx配置

```
  client_max_body_size 500M;
  client_header_buffer_size 51200k;
  large_client_header_buffers 4 51200k;
```

**增加后端响应超时时长**

* 普通nginx配置

```
  proxy_connect_timeout 90;
  proxy_send_timeout 90;
  proxy_read_timeout 90;

```
**允许跨域&携带cookie**

* 普通nginx配置

```
  proxy_pass_header Set-Cookie;
  proxy_pass_header Host;
  proxy_pass_header Access-Control-Allow-Origin;
  proxy_pass_header Access-Control-Allow-Credentials;
  proxy_pass_header Access-Control-Allow-Headers;
  proxy_set_header Host $proxy_host;

```

**if条件判断**

* 普通nginx配置

```
 set $flag 0;
 if ( $host = "yiqi.test-saas.bitnei.cn" ) {
 	set $flag "${flag}1";
 }
 if ( $request_uri ~ "new_logo.png" ) {
    set $flag "${flag}2";
 }
 if ( $request_uri ~ "bitbugfavicon.ico" ) {
    set $flag "${flag}3";
 }
 if ( $flag = "012" ) {
 	rewrite ~* /assets/yiqi/new_logo.png break;
 }
 if ( $flag = "013" ) {
    rewrite ~* /assets/yiqi/bitbugfavicon.ico break;
}

```

**二级目录配置**

* 普通nginx配置

```
  location ^~ /bit {
     root /opt/static/;
  }

```

**添加header**

* 普通nginx配置

```
  location /xx/{
     add_header 'Access-Control-Expose-Headers' 'to-sec,pub';
  }

```

**404跳转首页**

* 普通nginx配置

```
  location ^~ /bit {
     root /opt/static/;
     index index.html; 
     error_page  404  /index.html;
  }

```

# 总结

	以上为笔者在开发中,总结的配置,希望对读者有一定的帮助.
