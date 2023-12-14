---
title: 'RHEL 系安装 Mastodon'
date: 2022-08-11T13:15:10+00:00
description: 本篇文档将介绍如何在RHEL系服务器中安装Mastodon。
summary: 本篇文档将介绍如何在RHEL系服务器中安装Mastodon。
categories:
  - 技术分享
tags:
  - almalinux
  - centos
  - mastodon
  - rhel
  - rockylinux
  - sysadmin
keywords:
  - almalinux
  - centos
  - mastodon
  - rhel
  - rockylinux
  - sysadmin
feature: feature.png
featureAlt: Mastodon快速上手页截图 
images:
  - feature.png
slug: install-mastodon-on-rhel
type: posts
showComments: true
isCJKLanguage: true
draft: false
showTableOfContents: true
author: holger
---
{{< alert "circle-info" >}}
原始教程来自: [https://lala.im/4286.html](https://lala.im/4286.html)
{{< /alert >}}

## 前言 

本篇文档将介绍如何在RHEL系服务器中安装Mastodon。由于官方的文档只覆盖了Debian系安装，故本文作为一个补充，大部分操作流程应参考[官方安装文档](https://docs.joinmastodon.org/admin/install/)。

如果你还不知道Mastodon是什么，欢迎阅读: [欢迎加入长毛象！](https://holger.one/posts/welcome-to-mastodon/)</a>

本次安装实质性安装于AlmaLinux 9.0 (RHEL 9)。

## 准备工作 

  1. 能有公网访问权限并可被公网访问到的服务器；
  2. 一个能够长久使用的域名(域名是Mastodon实例的唯一标识，一旦确认不可更改)。

## 开始部署 
### 安装依赖 

```bash
dnf -y install epel-release // 安装扩展软件库
dnf -y groupinstall "Development Tools" // 安装编译所需依赖
dnf config-manager --set-enabled crb // RHEL 8无需此行
dnf -y install wget curl git openssl-devel readline-devel libicu-devel libidn-devel postgresql-devel protobuf-devel libxml2-devel libxslt-devel ncurses-devel sqlite-devel gdbm-devel zlib-devel libffi-devel libyaml-devel nscd jemalloc perl perl-core perl-base jemalloc-devel
```

### 安装NodeJS+Yarn 

```bash
curl --silent --location https://rpm.nodesource.com/setup_16.x | sudo bash -
dnf -y install nodejs
curl --silent --location https://dl.yarnpkg.com/rpm/yarn.repo | sudo tee /etc/yum.repos.d/yarn.repo
dnf -y install yarn
```

### 安装Redis 

```bash
dnf install redis
systemctl enable --now redis // 设置Redis开机自启并启动
```

### [安装PostgreSQL](https://www.postgresql.org/download/linux/redhat/)

点击上方链接并选择操作系统安装↑

#### 初始化数据 

```bash
/usr/pgsql-14/bin/postgresql-14-setup initdb

systemctl enable --now postgresql-12 // 启动PostgreSQL

sudo -u postgres psql // 登录PostgreSQL

CREATE USER mastodon CREATEDB; // 创建数据库
\q
```

### 安装ImageMagick 

```bash
dnf -y install ImageMagick ImageMagick-devel ImageMagick-perl
```

### 安装FFMPEG 

```bash
wget https://johnvansickle.com/ffmpeg/releases/ffmpeg-release-amd64-static.tar.xz
tar -xJf ffmpeg-release-amd64-static.tar.xz
cd ffmpeg-*-amd64-static
cp ffmpeg /usr/bin/ffmpeg
cp ffprobe /usr/bin/ffprobe
```

### 安装Nginx 

```bash
dnf -y install nginx
systemctl enable --now nginx
```

### 创建Mastodon用户 

```bash
useradd mastodon
passwd -l mastodon
su - mastodon
```

### [安装 Ruby](https://docs.joinmastodon.org/admin/install/) 

```bash
git clone https://github.com/rbenv/rbenv.git ~/.rbenv
cd ~/.rbenv && src/configure && make -C src
echo 'export PATH="$HOME/.rbenv/bin:$PATH"' &gt;&gt; ~/.bashrc
echo 'eval "$(rbenv init -)"' &gt;&gt; ~/.bashrc
exec bash
git clone https://github.com/rbenv/ruby-build.git ~/.rbenv/plugins/ruby-build
RUBY_CONFIGURE_OPTS=--with-jemalloc rbenv install 3.0.3
rbenv global 3.0.3
gem install bundler --no-document
```

### [安装Mastodon本体](https://docs.joinmastodon.org/admin/install/)

```bash
git clone https://github.com/tootsuite/mastodon.git live && &lt;em>cd&lt;/em> live
git checkout $(git tag -l | grep -v 'rc&#91;0-9]*$' | sort -V | tail -n 1)
bundle config deployment 'true'
bundle config without 'development test'
bundle install -j$(getconf _NPROCESSORS_ONLN)
yarn install --pure-lockfile
```

### 初始化站点 

```bash
RAILS_ENV=production bundle exec rake mastodon:setup
```

### 配置Nginx与Systemd 

```bash
exit // 进入root用户

curl https://get.acme.sh | sh -s email=mastodon@example.com 

acme.sh  --issue  -d example.com  --nginx

mkdir /etc/nginx/ssl

cp /home/mastodon/live/dist/nginx.conf /etc/nginx/conf.d/mastodon.conf 

vim /etc/nginx/conf.d/mastodon.conf //修改证书地址为下方acme安装证书到的位置

acme.sh --install-cert -d example.com \
    --key-file       /etc/nginx/ssl/mastodon.pem  \
    --fullchain-file /etc/nginx/ssl/mastodon.pem \
    --reloadcmd     "service nginx force-reload"

cp /home/mastodon/live/dist/mastodon-*.service /etc/systemd/system/

systemctl daemon-reload
systemctl enable --now mastodon-web mastodon-sidekiq mastodon-streaming
```

## 完成 

大功告成！现在便可以用浏览器打开你的域名访问Mastodon啦~ 

{{< alert >}}
注： 本篇文档并不涉及任何系统加固、防火墙配置等内容。如果你尚未对服务器进行安全加固，请不要将其运作在生产环境中。
{{< /alert >}}