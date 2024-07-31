---
slug: nginx-for-openwrt
title: 为 Openwrt 编译 Nginx
date: 2024-01-19 15:42:29
tags: 
  - Network
  - Nginx
  - Openwrt
categories: 
  - 记录
summary: 使用 musl 编译 Nginx
---
# 起因
由于 openwrt 使用 musl-gcc，无法运行 gcc 编译的程序，需要额外编译

并且 openwrt 官方编译的 ipk 自由度不高，需要自己编译功能

# 准备工作
参见上篇：[编译 Nginx 并开启 HTTP/3](/posts/nginx-with-http3)

# 编译 musl-gcc
1. [musl 源码](https://musl.libc.org/)，解压 tar.gz，目录：`$WORKER/musl`
2. `cd ./musl` `./configure` `make` `sudo make install`
3. `/usr/local/musl/bin/musl-gcc -v`确认安装完毕
4. `export CC=/usr/local/musl/bin/musl-gcc`，将编译器设为 musl-gcc

# 开始编译
1. 进入 nginx 目录`cd ./nginx`
2. 配置：
```bash
./configure \
    --with-compat  \
    --with-http_ssl_module  \
    --with-http_v2_module  \
    --with-http_v3_module  \
    --with-http_realip_module  \
    --with-http_addition_module \
    --with-http_sub_module \
    --with-http_dav_module \
    --with-threads  \
    --with-pcre=../pcre  \
    --with-pcre-jit  \
    --with-zlib=../zlib  \
    --with-openssl=../libressl \
    --conf-path=/etc/nginx/nginx.conf \
    --prefix=/var/nginx \
    --pid-path=/var/run/nginx.pid \
    --user=root
```
3. `make`，编译结果会存于`$WORKER/nginx/objs`
4. 安装（不必须）：`sudo make install`

# 注意
添加必要文件夹，如：/var/nginx/logs