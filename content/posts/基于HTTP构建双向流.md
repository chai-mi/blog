---
slug: stream-over-http
title: 基于HTTP构建双向流
date: 2024-09-09 12:37:32
tags: 
  - Network
  - HTTP
categories: 
  - protocol
summary: 一种使用HTTP产生双向流的思路
---
# 背景
HTTP 作为最流行的应用层协议，无疑展现了其强大的生命力，无数的基础设施优先为 HTTP 服务，各类编程语言也纷纷具有最完善的框架支持

然而，request/response 模型注定了 HTTP 的通信双方不是对等的。server 总是要在 client 发起 request 后才能回复 response，但是有时真的有双向流的通信需要，又不得不采取一些蹩脚的措施，比如 websocket，亦或是 CONNECT 方法

然而 Websocket 毕竟是另一种协议，CONNECT 方法也不被大多数 HTTP 基础设施接受，归根结底，还是需要**额外**支持，并不直接与 HTTP 兼容

# stream
如果熟悉 Fetch API，应该知道 request.body 和 response.body 实际都是 readablestream，这意味着它并不是一次性将 body 发送/接受的

并且 HTTP 并没有规定 server 必须完全读取 request 后才能进行 response 响应，事实上，使用 POST/PUT 传递大文件的时候，server 总是在完全上传完毕前进行了 200 响应

那么 client 只需将 request.body 设为 readablestream，并且 server 也将 response.body 设为 readablestream，如此，client 可以在任意时刻写入数据到 request.body，server 也可以在任意时刻写入数据到 response.body，而在另一端随时读取，便可构建基于 HTTP 的双向流

# 基础设施兼容性
### 反向代理软件
Nginx 显然是毫无疑问支持的，只需进行如下设置
```
proxy_pass http://you-server;
proxy_http_version 1.1;
proxy_request_buffering off;
```
> 由于 HTTP/1.0 不支持流式传输，而 Nginx 反向代理默认使用 HTTP/1.0，因此需要设置为 1.1

### CDN
[Cloudflare 无法建立双向流](https://community.cloudflare.com/t/streaming-a-request/14724)，因为它无论如何都会缓冲 request.body，在完全读取 body 之前，不会向源服务器发送请求。

这源于一种叫 [slowloris](https://en.wikipedia.org/wiki/Slowloris_(computer_security)) 攻击，绝大多数 CDN 为了缓解该攻击而对请求主体进行缓冲，以至于无法通过 CDN 建立双向流

> 我并没有测试太多 CDN，也许有的 CDN 忽视了这一问题

### 编程语言
大多数编程语言的 HTTP 标准库都将 request.body/response.body 视作一个 IO 接口，完全可以流式传输

~~并且自己实现一个流式请求的 HTTP/1.1 封装库并不难~~

### 浏览器
如上所说，Fetch API 已实现流式传输，自然浏览器不存在兼容性问题

# 对比其他 Web 双向流的方案
### CONNECT
CONNECT 方法虽然是标准的 HTTP 方法，但是常常作为**拓展**出现，与原生的 HTTP 有较大区别

### websocket
Websocket 并不属于 HTTP，尽管它常常依托 HTTP upgrade 建立连接，总而言之，它是**另一种协议**而非 HTTP，支持它需要额外实现

### grpc
grpc 为流式接口考虑了很多，但总而言之，它也只是基于 HTTP/2 的**另一种协议**，支持它需要额外实现

~~比如 Nginx 到现在也不支持 HTTP/2 回源，但额外支持了 grpc_pass~~

### webtransport
基于 HTTP/3 的新流式协议，但仍处于草案阶段，基础设施大量不足

# 总结
使用尽可能原生的 HTTP 总是有独特的优势，绝大多数 HTTP 基础设施都是将它视为一个简单的 HTTP 请求，并且可以随意使用 HTTP/1.1，HTTP/2，HTTP/3