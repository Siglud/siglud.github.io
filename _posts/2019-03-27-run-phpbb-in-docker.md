---
layout: post
title:  "在Docker中运行phpBB"
date:   2019-03-27 20:12:00 +0800
author: Siglud
categories:
  - PHP
tags:
  - PHP
  - Nginx
comment: true
share: true
---
因为工作上的需要，需要搞个临时的PHPBB论坛，考虑直接在Docker上直接跑了。

首先运行一个MySQL

```
docker run --restart=always --name mysql -v /etc/mysql/conf.d:/etc/mysql/conf.d -e MYSQL_ROOT_PASSWORD=root -v /var/lib/mysql:/var/lib/mysql -p 3306:3306 -d mysql:8
```

直接拿一个MySQL8在把端口运行在本地了

然后开启PHP-FPM

```
docker run -d --restart=always --name phpfpm -p 9000:9000 -v /data/web_root:/app bitnami/php-fpm
```

/data/web_root 是本地的PHPBB放置的位置

开启Nginx
```
docker run -d --restart=always --name myNginx -p 80:80 -v /data/web_root:/app -v /data/nginx_conf:/etc/nginx/conf.d nginx
```

为了方便处理，把nginx的WebRoot放到和PHP-FPM同样的位置，同时把nginx的配置文件映射出来方便修改，否则需要不停的在镜像里面cp来cp去麻烦，也不方便调试和修改

在/data/nginx_conf下建立一个default.conf

```
server {
    listen       80;

    root /app;
    index  index.php index.html index.htm;

    location / {
        try_files $uri $uri/ @rewriteapp;
        index  index.php index.html index.htm;
    }

    location @rewriteapp {
        rewrite ^(.*)$ /app.php/$1 last;
    }

    location /install/app.php {
        try_files $uri $uri/ /install/app.php?$query_string;
    }

    location ~ /(config\.php|common\.php|cache|files|images/avatars/upload|includes|(?<!ext/)phpbb|store|vendor) {
        deny all;
        internal;
    }

    location ~ \.php(/|$) {
        fastcgi_index            index.php;
        fastcgi_pass   172.17.0.1:9000;
        fastcgi_param SCRIPT_FILENAME /app$fastcgi_script_name;
        include                  fastcgi_params;
    }
}
```
相比原版的，只改了fastcgi_pass的设置方式，因为我们只能走sockets。

如果开了selunix，记得关闭它
```bash
setenforce 0
```
把论坛的部分目录设置为777权限
```
cd /data/web_root && chmod 777 -R cache files store images/avatars/upload
```
因为我用的PHPBB版本（3.2.5）当前其实对MySQL8兼容得不好，主要表现在MySQL无法登陆，临时解决方案就是运行下SQL修改一下ROOT的密码验证方式

```sql
use mysql;
ALTER USER 'root'@'%' IDENTIFIED WITH mysql_native_password BY 'root';
FLUSH PRIVILEGES;
```
然后一路安装即可