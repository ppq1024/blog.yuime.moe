---
date: 2026-05-29
name: site-migration
title: 关于最近的站点迁移
draft: false
tags:
  - 杂谈
  - Nginx
---

前两天把本站从 GitHub Pages 迁移到了个人服务器上，顺便修了一波 bug 并进行勘误。要说最根本的动机还是 GitHub Action 构建流程耗时太长，每次都要重新建立构建环境，Hugo 快速构建的优势完全没有发挥出来，同时 GitHub Pages 也不能像 Nginx 那样进行更深层次的定制（突然想起来之前摸过的 i18n for 404，好像可以再重新加回来了，~~虽然本站大部分内容都没有被翻译~~）。

大部分技术框架都没有变，只是前端服务从 GitHub Pages 迁移到了 Nginx，服务器上用 Nginx 跑了些私有服务的反向代理，就顺便把本站也搬过来了。个人服务的私有性基于 mTLS + 私有 CA，这两天应该会单独写一篇讲一下。

## 关于站点配置

迁移后需要解决两个问题：一是证书配置，二是自动化构建部署流程。个人服务一开始打算用私有证书，之前在进行内网服务开发的时候给 *.internal 域名签过一套（私有域名本身也不可能拿到公共 CA 签名），直接把脚本搬过去就可以用。但考虑到域名下可能存在公开服务，最后就选择了 Let's Encrypt + certbot 的方案。

### 配置服务端证书

首先需要确保域名已解析到服务器 IP，由于修改记录可能需要等待一段时间生效（主要是等待 DNS 间同步），建议这一步在环境准备前进行，在等待生效的过程中再去配置环境。

我的服务器环境是 Ubuntu 24.04.4 LTS，配置过程以此为例，其他发行版可能存在细微差异。

安装 certbot 和 nginx 插件：

```bash
sudo apt update
sudo apt install certbot python3-certbot-nginx
```

安装证书前先完成 Nginx http 配置，安装证书时插件会自动配置 https。

{{< notice note >}}
插件仅会修改 sites-available 下的配置，不会处理 conf.d 下的内容。
{{< /notice >}}

```ini
server {
    listen 80;
    server_name blog.yuime.moe;
    
    index index.html index.htm;

    location / {
        alias /var/www/blog/;
        autoindex on;
    }
}
```

执行 certbot 安装证书：

```bash
sudo certbot --nginx -d blog.yuime.moe
```
成功后插件会将 Nginx 配置修改为：

```ini
server {
    server_name blog.yuime.moe;

    index index.html index.htm;

    location / {
        alias /var/www/blog/;
        autoindex on;
    }

    listen 443 ssl; # managed by Certbot
    ssl_certificate /etc/letsencrypt/live/blog.yuime.moe/fullchain.pem; # managed by Certbot
    ssl_certificate_key /etc/letsencrypt/live/blog.yuime.moe/privkey.pem; # managed by Certbot
    include /etc/letsencrypt/options-ssl-nginx.conf; # managed by Certbot
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem; # managed by Certbot
}
server {
    if ($host = blog.yuime.moe) {
        return 301 https://$host$request_uri;
    } # managed by Certbot

    listen 80;
    server_name blog.yuime.moe;
    return 404; # managed by Certbot
}
```

重新导入 Nginx 配置：

```bash
sudo systemctl reload nginx
```

### 自动化部署

之前的方案是使用 GitHub Action，在 push 到主分支时自动构建并部署到 GitHub Pages 上，每次都要重新建立构建环境并重头构建，耗时大概要 40-50s。一开始的想法是让 GitHub Action 仅通知服务器进行更新，服务器收到通知后自主从 GitHub 拉取最新代码并构建部署，就去用 Go 手搓了一个 [简单后端](https://github.com/ppq1024/blog-sync)。

不过后来测试的时候发现服务从 GitHub 拉取代码耗时非常长，和 GitHub Action 构建流程耗时差不多。既然服务器对 PC 来说是可达的，那为什么不直接从 PC 推送代码到服务器上呢？PC 推送代码后再通知服务器构建部署，这样以来也不需要暴露公共服务，还能进一步降低攻击面。

服务仅负责单一职责，本身可以做到非常简单，在设计过程中仅监听环回地址，不处理 TLS 和复杂路由，这部分由 Nginx 反向代理，只有我自己签署的客户端证书才放行，认证失败或不提供证书则直接返回 403。。

```ini
server {
    server_name service.yuime.moe;

    index index.html index.htm;
    
    location = /action/sync/blog {
	if ($ssl_client_verify != "SUCCESS") {
            return 403;
        }
        proxy_pass http://[::1]:8000;
    }

    # ... Other service

    listen 443 ssl; # managed by Certbot
    ssl_certificate /etc/letsencrypt/live/service.yuime.moe/fullchain.pem; # managed by Certbot
    ssl_certificate_key /etc/letsencrypt/live/service.yuime.moe/privkey.pem; # managed by Certbot
    include /etc/letsencrypt/options-ssl-nginx.conf; # managed by Certbot
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem; # managed by Certbot
    ssl_client_certificate /etc/nginx/ssl/client.ppq.ca.chain.pem;
    ssl_verify_client optional;
}
server {
    if ($host = service.yuime.moe) {
        return 301 https://$host$request_uri;
    } # managed by Certbot
    listen 80;

    server_name service.yuime.moe;
    return 404; # managed by Certbot
}

```

然后将推送和部署流程整理为一个脚本：
```bash
#! /usr/bin/bash

git push vps main && curl --cert ~/.ssl/ppqarch.crt --cert-type DER --key ~/.ssl/ppqarch.key --key-type DER https://service.yuime.moe/action/sync/blog
```

目前仅更新主题仓库时需要到服务器手动拉取一下，后续会考虑自动拉取。

### HTTP or SSH

目前的同步服务实际只执行了一条命令，即 `hugo --minify --destination ${output_dir} --cleanDestinationDir`，那为什么不直接用 ssh 执行呢？

虽然有一部分"服务搓都搓了就继续用吧"的历史原因（~~这一天都不到叫什么历史~~），但更重要的是最小化权限原则，同时服务本身的扩展成本也非常低，像权限控制、日志之类的功能实现起来比 ssh 执行命令或脚本方便的多，通过添加安全中间件也可以实现更细粒度的权限控制。而 ssh 暴露的权限过高，尽管目前的流程仍需要 ssh 权限进行代码推送，但这一步本身可以从流程里解耦出来并迁移到更安全的解决方案。

## 一些杂谈

尽管便捷性在开发过程中还是很重要的，但对于本站甚至其他个人项目来说永远不是第一优先级。对于个人项目来说开发者同时作为甲方和乙方，我自己比较喜欢折腾，因此定制化本身就是甲方的最大需求，这也是很多技术人员喜欢重复造轮子的原因。而且折腾技术本身就是一个学习和自我提升的过程，作为从前 AI 时代走过来的程序员，深知技术的价值远大于代码本身，AI 编程降低了初学者的门槛，但对于高级工程师来说只是更多的经历从编码转向了架构设计。

之前的杂谈里也提到过我租过一个阿里云的 ECS，但本站没有考虑部署在那边，主要还是因为境内站点需要 ICP 备案，但个人网站可能会出现频繁变更，也没有面向公共的商业服务，因此部署在境外更加灵活，也免去了备案的麻烦。
