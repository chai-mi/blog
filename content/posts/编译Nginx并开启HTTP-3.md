---
title: 编译 Nginx 并开启 HTTP/3
date: 2024-01-19 15:13:32
tags: 
  - Network
  - Nginx
categories: 
  - 记录
url: /posts/nginx-with-http3
---
# 准备工作
1. [Nginx 源码](https://nginx.org/en/download.html)，解压 tar.gz，目录：`$WORKER/nginx`
2. [pcre 源码](https://github.com/PCRE2Project/pcre2/releases)，解压 tar.gz，目录：`$WORKER/pcre`
> 该库是`location`指令和`ngx_http_rewrite_module`模块中正则表达式支持所必需的
3. [zlib 源码](https://zlib.net)，解压 tar.gz，目录：`$WORKER/zlib`
> 该库是`ngx_http_gzip_module`模块所必需的
4. [libressl 源码](https://ftp.openbsd.org/pub/OpenBSD/LibreSSL/)，解压 tar.gz，目录：`$WORKER/libressl`
> 该库用于替代`openssl`库，它比`openssl`有更好的 QUIC 支持

# 开始编译
1. 进入 nginx 目录`cd ./nginx`
2. 配置：
```bash
./configure \
    --with-compat  \
    --with-file-aio  \
    --with-http_ssl_module  \
    --with-http_realip_module  \
    --with-http_v2_module  \
    --with-http_v3_module  \
    --with-stream  \
    --with-stream_realip_module  \
    --with-stream_ssl_module  \
    --with-stream_ssl_preread_module  \
    --with-threads  \
    --with-pcre=../pcre  \
    --with-pcre-jit  \
    --with-zlib=../zlib  \
    --with-openssl=../libressl
```
3. `make`，编译结果会存于`$WORKER/nginx/objs`
4. 安装（不必须）：`sudo make install`