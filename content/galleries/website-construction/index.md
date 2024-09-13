---
title: "记录网站搭建的一些杂事"
date: "2024-09-12T10:57:15+08:00"
draft: false
categories:
  - fun
tags:
  - blog
  - website
series:
  - "没什么用的记录"
---

## 一、简介

这篇文章主要记录搭建博客以及其他网站时用到的一些内容，包括：

- 获取域名
- 配置 Cloudfla DNS 解析
- 配置 Https
- 可能的 Nginx 配置

## 二、获取域名

学生获取域名可以前往[Github 教育认证](https://github.com/education)，完成教育认证后可以在[教育优惠包](https://education.github.com/pack)中找到[".TECH domain free for 1 year"](https://get.tech/github-student-developer-pack?utm_source=Github&utm_medium=Partnership&utm_campaign=Brands&utm_term=Partnership&utm_content=Github)，可以在里面领取为期一年的免费域名。

![.TECH 域名](./images/Tech%20Domain.png)

## DNS 域名解析

[Cloudflare](https://www.cloudflare.com/)提供了免费的 DNS 解析服务和代理服务。免费计划的代理印象比较深的一个限制是单个文件上传大小限制在 100MB，如果要搭建[Alist](https://alist.nn.ci/)网盘服务的话，可能要注意关闭 Cloudflare 的代理，让其只解析 DNS。

## 配置 HTTPS

搭建网站一般都需要配置 HTTPS，但是大部分 HTTPS 证书的申请都需要付费，自签名的证书又没有 CA 认证，会有一些问题。白嫖的方案是使用[`Let's EnCrypt`](https://letsencrypt.org/)，提供免费的 ssl 证书，而且也是被认证的，不会有签名问题。它使用一个名为 Cerbot 的客户端来访问，可以访问[Cerbot 官网查看](https://certbot.eff.org)，linux 下可以使用`pip`或`snap`安装命令行客户端，具体参见[官方教程](https://certbot.eff.org/instructions)

## Nginx 配置

Nginx 新建一个网站配置，可以在`/etc/nginx/conf.d/`路径下新建一个`<filename>.conf`的文件，文件可以按自己喜好命名。下面是一个样例的配置

```conf
# /etc/niginx/conf.d/web.conf
server {
        # 设置nginx https监听的端口为443
        listen  443 ssl;
        # 设置域名，your.domain.com应该是你的网站
        server_name your.domain.com;
        # 证书文件路径
        ssl_certificate         /path/to/your/certificate;
        # 证书私钥文件路径
        ssl_certificate_key     /path/to/yout/privatekey;
        # ssl验证配置
        ssl_session_timeout     5m;
        # 安全链接可选加密协议
        ssl_protocols TLSv1 TLSv1.1 TLSv1.2 TLSv1.3;
        # 配置加密套件
        ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:HIGH:!aNULL:!MD5:!RC4:!DHE;
        # 使用服务器端首选算法
        ssl_prefer_server_ciphers on;

        error_page      404             /404.html;
        error_page      500 502 503 504 /50x.html;

        # 错误网页
        location = /50x.html {
                root html;
        }

        # 一个http代理的配置
        # location = / {
        #         proxy_pass http://127.0.0.1:10086;
        #         proxy_intercept_errors on;
        #         error_page 400 = https://跳转的域名;

        #         # 代理
        #         proxy_http_version 1.1;
        #         proxy_set_header Upgrade $http_upgrade;
        #         proxy_set_header Connection "upgrade";
        #         proxy_set_header Host $http_host;
        #         proxy_read_timeout 600s;
        #         # 传递客户端 IP 地址
        #         proxy_set_header X-Real-IP $remote_addr;
        #         proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        # }

        # 一个alist的配置
        location / {
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            proxy_set_header Host $http_host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header Range $http_range;
                  proxy_set_header If-Range $http_if_range;
            proxy_redirect off;
            # 此处应该为alist监听的网页和端口
            proxy_pass http://127.0.0.1:9091;
            client_body_timeout 600s;
            proxy_connect_timeout 600s;
            proxy_read_timeout 600s;
            proxy_send_timeout 600s;
            # the max size of file to upload
            client_max_body_size 4000m;
        }
}

```
