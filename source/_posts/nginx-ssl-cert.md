---
title: nginx配置ssl实现https/http2访问
tags:
  - nginx
categories: Tools
date: 2019-08-14 23:26:32
---


最近我在vultr上的买的VPS的IP被墙掉了，导致之前在VPS上部署的一个gitbook网页访问不了。因此，不得不将这个网页迁移到我的阿里云机器上部署，这里记录下 nginx 的 SSL 配置。

### 0x01 域名 & 证书

如果在阿里云上购买了域名，应该是能够直接申请一个免费的证书的，证书申请完成后，直接下载下来拷贝到服务器上。可以选择Nginx的模板，下载下来的是一个`.pem`扩展名的证书文件和一个`.key`扩展名的私钥文件。

### 0x02 nginx 配置

这里将证书和私钥文件放在自己创建的 `/cert/` 目录下，并重命名了。

接下来就是修改 nginx 的配置文件了

```conf
server {
    listen 443 ssl default_server;
    listen [::]:443 ssl default_server;

    ssl on;
    ssl_certificate   /root/cert/a.pem;
    ssl_certificate_key  /root/cert/a.key;
    ssl_session_timeout 5m;
    ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:ECDHE:ECDH:AES:HIGH:!NULL:!aNULL:!MD5:!ADH:!RC4;
    ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
    ssl_prefer_server_ciphers on;

    location / {
        root html;
        index index.html index.htm;
    }
}
```

重启 nginx

```shell
systemctl restart nginx
```

### 0x03 后续配置

nginx 起好了之后，还需要在阿里云的控制台修改域名解析规则，将域名解析到新的服务器IP上。此外，阿里云的机器还需要修改安全组配置，添加 443 端口的ACL规则，这样这个机器才是可达的。

### 0x04 HTTP 2

等弄好了，过了几分钟我才记起来 nginx 现在使能了 SSL 的前提下可以直接使能 HTTP 2 了，索性试试吧。

再次修改 nginx 的配置，直接在原来的 listen 配置加上 http2 即可

```conf
listen 443 ssl http2 default_server;
listen [::]:443 ssl http2 default_server;
```

重启 nginx

```shell
systemctl restart nginx
```

打开浏览器的控制台，重新访问网页，可以看到 HTTP 2 是否使能成功了(需要在Chrome developer tool里边的network加上Protocol一栏)。

{% asset_img chrome_h2.png %}