---
slug: hello-ech
title: Hello! ECH
date: 2024-09-14 13:23:07
tags:
  - Network
  - HTTP
categories:
  - protocol
summary: 网络隐私的最后一块拼图
---
最近 cloudflare 重新启用了 [ECH](https://developers.cloudflare.com/ssl/edge-certificates/ech) (Encrypted Client Hello)，本博客完全托管在 cloudflare pages 上，目前已可以通过 ECH 访问，你可以点击[这里](/cdn-cgi/trace)查看。

如果你能看到 `sni=encrypted`，恭喜你，你已经通过 ECH 访问本站！

如果你完全不懂 ECH ，或者在上述测试结果是 `sni=plaintext`，那么继续读下去吧，我将介绍 ECH 为什么重要，以及如何配置你的网络环境以通过 ECH 保护你的上网隐私

# 什么是 ECH
### HTTP
HTTP 作为最广泛运用的应用层协议，如今大部分上网行为都依托于 HTTP 框架。但是 HTTP 是明文协议，中间人能够一览无余

设想你使用校园网登录你的 b 站账号，并点开舞蹈区准备欣赏一下舞蹈，而这一切都能被学校知道得清清楚楚；即使使用移动网络绕过学校的监管，你是否又能接受 ISP 对你进行可能的监控？

好在目前绝大多数上网行为都使用 HTTPS，它**完全加密**了所有 HTTP 数据，保证你上网免受中间人的窥视

不过 HTTPS 本质上只是 HTTP over TLS，所以我们接下来聊一聊 TLS

### TLS
理论上，TLS 已经保证你的数据不受窥视、篡改，但是 TLS 依然暴露了少数敏感数据

当然这里还是有很大不同，没有 TLS，你上网的一举一动都完全会被中间人知晓。譬如，只有 HTTP，学校能完全知道你点开了哪个视频的 BV 号，看的视频的 TAG 等等，但有 TLS，学校就只能知道你访问 www.bilibili.com，你可以坚称自己只是点开了宋浩的视频

好吧，那么 TLS 到底暴露了什么敏感数据呢，其中最重要的便是 SNI (Server Name Indication)

在 HTTP 中，我们有 Host 指示服务器，这是实现虚拟主机的必要条件，有了虚拟主机，使用单个 IP 建立多个 Web 便成了可能

然而，TLS 会在 HTTP 开始传输前握手，所以 TLS 在握手会附带 SNI 拓展以实现相同功能，而 SNI 会以明文形式出现在 TLS handshake 中

尽管如此，SNI 相对于真正的用户数据并没有那么敏感，所以 TLS 广泛应用于各种应用层协议中

### ECH
终于到了本篇文章的主角：ECH。如上文所说，TLS 解决安全性问题，但中间人仍可根据 SNI 实现隐私收集或网络审查

譬如，你或许不在乎学校发现你访问 www.bilibili.com，但你也不想访问 pornhub.com 被学校知道吧；深信服或 GFW 也仍可依据 SNI 实现网络封锁

这便是 ECH 要做到的事，但是实现加密的前提是有共同密码 (key)，在 TLS handshake 前，双方并没有进行任何数据传输，自然也无法享有共享秘密

然而在一切网络传输发生前，首先会进行 DNS 查询，设想服务器将公钥放置于 DNS 记录中，客户端就使用该公钥对 SNI 等敏感拓展进行加密

但是，DNS 是明文协议，中间人仍可以伪造公钥实现攻击。幸运的是，当 ECH 被人们重视的时期，加密 DNS 已并不稀奇，使用加密 DNS 查询即可避免

# cloudflare 如何实现 ECH 保护隐私
参考 [cloudflare 博客](https://blog.cloudflare.com/encrypted-client-hello)

# ECH 安全性细节
### 中间件兼容性
待续

### 避免降级攻击
待续

### 公钥过期/无效
待续

# 如何设置我的网络以使用 ECH 保护隐私
待续