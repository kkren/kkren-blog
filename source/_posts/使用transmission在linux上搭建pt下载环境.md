---
title: 使用transmission在linux上搭建pt下载环境
date: 2018-05-26 20:43:10
urlname: transmission-in-linux
tags: 技术
typora-copy-images-to: ..\images
---
现在在linux服务器上部署的pt下载器主要是aria2， transmission和rtorrent，然而aria2要下载pt的话还需要伪装客户端，rtorrent的webui太丑加载速度还很慢，于是最后选择了transmission。博客中提到的服务器的配置为Intel Atom N2800 2c/4 t 1.86GHz，4Gb DDR3 1066，2T HDD， 位于欧洲，不抗投诉。<!-- more -->

安装transmission与deamon
----
服务器的系统环境是debian9，所以可以很方便用包管理器安装需要的软件

```shell
apt -y install transmission transmission-deamon
systemctl enable transmission-deamon
```

好像没有什么特别的配置要配置的，反正webui里都可以改

安装修改版的transmission webui
----
[项目地址](https://github.com/ronggang/transmission-web-control)，安装过程也是全自动的

```shell
wget https://github.com/ronggang/transmission-web-control/raw/master/release/install-tr-control.sh
bash install-tr-control.sh
```

接下来输入1然后按回车就可以了</br>然后重启下transmission

```shell
systemctl restart transmission-deamon
```

## 安装 transmission-rss



使用nginx 反代webui
----

transmission的webui没有ssl也没法加密码，所以用nginx反代下，性能更高。我安装的nginx是来自[烧饼博客上的教程](https://sb.sb/blog/debian-install-nginx-php-mysql/)，下面是配置文件
```
    server {
    listen 80 default_server;
    listen [::]:80 default_server;
    server_name example.com;
    
    location /.well-known/acme-challenge {
    root /var/www/letsencrypt;
    }
    
    location / {
    return 301 https://$host$request_uri;
    }
    }
    server {
    listen 443 ssl http2 default_server;
    listen [::]:443 ssl http2 default_server;
    
    server_name example.com;
    client_max_body_size 10m;
    server_tokens off;
    
    root /var/www/rutorrent;
    index index.html index.htm index.php;
    
    ssl_dhparam /etc/nginx/ssl/dhparam.pem;
    
    ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
    ssl_ciphers 'ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-AES128-SHA256:ECDHE-RSA-AES128-SHA256:ECDHE-ECDSA-AES128-SHA:ECDHE-RSA-AES256-SHA384:ECDHE-RSA-AES128-SHA:ECDHE-ECDSA-AES256-SHA384:ECDHE-ECDSA-AES256-SHA:ECDHE-RSA-AES256-SHA:DHE-RSA-AES128-SHA256:DHE-RSA-AES128-SHA:DHE-RSA-AES256-SHA256:DHE-RSA-AES256-SHA:ECDHE-ECDSA-DES-CBC3-SHA:ECDHE-RSA-DES-CBC3-SHA:EDH-RSA-DES-CBC3-SHA:AES128-GCM-SHA256:AES256-GCM-SHA384:AES128-SHA256:AES256-SHA256:AES128-SHA:AES256-SHA:DES-CBC3-SHA:!DSS';
    ssl_prefer_server_ciphers on;
    
    ssl_session_cache shared:SSL:50m;
    ssl_session_timeout 1d;


    ssl_certificate /etc/nginx/ssl/fullchain.cer;
    ssl_certificate_key /etc/nginx/ssl/dl.key;


    add_header Strict-Transport-Security "max-age=63072000; includeSubDomains; preload";
    add_header X-Frame-Options SAMEORIGIN;
    add_header X-Content-Type-Options nosniff;
    add_header X-XSS-Protection "1; mode=block";
    
    location / {
    auth_basic "Restricted";
    auth_basic_user_file /etc/nginx/passwd;
    proxy_pass  http://127.0.0.1:9091;
    }

    location ~ \.php$ {
    fastcgi_pass unix:/run/php/php7.2-fpm.sock;
    include snippets/fastcgi-php.conf;
    }
    
    access_log /var/log/nginx/rutorrent.access.log;
    error_log /var/log/nginx/rutorrent.error.log;
    }
```
其中，`/etc/nginx/passwd`的生成方法是`printf "user:$(openssl passwd -crypt password)\n" >> /etc/nginx/passwd`。

最后的效果
----
![1540133454051](https://i.loli.net/2018/10/22/5bcda5ea4387b.jpg)

真香