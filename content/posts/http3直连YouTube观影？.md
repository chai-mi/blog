---
slug: watch-youtube-through-http3
title: http3 直连 YouTube 观影？
date: 2023-08-29 19:36:29
tags:
  - Network
  - QUIC
categories: 
  - 记录
summary: form end user
---
# 前言
老实说，我本不打算写这篇博客的：因为没有推广的价值，它本质是利用了 GFW 的一个漏洞，如果这漏洞被修复，该方法自然失效

但是我在谷歌对应资料的时候，意外地了解到这个“漏洞”[历史意外的悠久？](https://www.v2ex.com/t/370189?p=1)，居然能追溯到 2017 年。看来此方法可能能使用很长一段时间，因此写下此文。~~反正我的博客也没什么人看~~
# 原理
要说这个漏洞也简单，GFW 没有阻断 QUIC 流量的 YouTube 视频流的域名(googlevideo.com)，根据 GFW 目前的手段来看，此域名处于如下状态：

| DNS 污染 | 路由黑洞 | TLS SNI 阻断 | QUIC SNI 阻断 |
| -------- | -------- | ------------ | ------------- |
| yes      | no       | yes          | no            |

> 补充：该域名下部分 ipv4 是路由黑洞，建议 DNS 模块只返回 ipv6 的地址

这就给了我们钻漏洞的机会：
1. 开启浏览器的 QUIC 支持
2. 使用未被污染的 DNS
这样视频流的流量就会通过 QUIC 直连到 YouTube 的 CDN

不过很遗憾，普通的 HTTP 代理或者 socks 代理均不支持代理 QUIC 流量，当浏览器启用了代理，就不会尝试使用 QUIC 进行连接，因此浏览器一定要保持没有代理的状态

不过浏览器不代理，显然是打不开 YouTube 的，那视频流也无从谈起了，这样变成了先有鸡还是先有蛋的问题了。有没有一种情况能仅代理 `www.youtube.com` 而不代理 `*.googlevideo.com` 且不开启系统代理的呢

这就是透明代理了，在使用 tun/tproxy 的情况下，处理 DNS 请求并分流这两个域名，便可实现
# 操作方法
## 启用 QUIC
如果你使用 edge 浏览器，在地址栏输入 [edge://flags](edge://flags)，并在随后出现页面的搜索栏里键入 `quic`，找到 `Experimental QUIC protocol` 项设置为 `Enabled` 并重启浏览器
如果你使用 chrome 浏览器，只需输入 [chrome://flags](chrome://flags)，其余操作与上述相同
## 使用未污染的 DNS
在使用 tun/tproxy 时，浏览器首先会发起 DNS 请求再开始连接，GFW 通过返回虚假的 IP 地址来阻断对被墙的域名的访问（DNS 污染）。通过 tun/tproxy 处理 DNS 请求，使其获取“干净”的 IP。可以考虑使用 DoT 或者 DoH，或者直接代理 DNS 请求，避免 GFW 污染 DNS 查询结果

以下是 sing-box 使用 tun 的示例：
```json
{
    "dns": {
        "servers": [
            {
                "tag": "fakeip",
                "address": "fakeip"
            },
            {
                "tag": "ali-dns",
                "address": "223.5.5.5",
                "detour": "direct-out"
            },
            {
                "tag": "google-dns",
                "address": "8.8.8.8",
                "detour": "proxy"
            },
            {
                "tag": "block-dns",
                "address": "rcode://success"
            }
        ],
        "rules": [
            {
                "outbound": "any",
                "server": "ali-dns"
            },
            {
                "geosite": "category-ads-all",
                "server": "block-dns"
            },
            {
                "geosite": "cn",
                "server": "ali-dns"
            },
            {
                "domain_suffix": ".googlevideo.com",
                "server": "google-dns"
            },
            {
                "query_type": [
                    "A",
                    "AAAA"
                ],
                "server": "fakeip"
            }
        ],
        "final": "ali-dns",
        "reverse_mapping": true,
        "fakeip": {
            "enabled": true,
            "inet4_range": "198.18.0.0/16",
            "inet6_range": "fc00::/18"
        }
    },
    "route": {
        "rules": [
            {
                "protocol": "dns",
                "outbound": "dns-out"
            }
        ],
        "final": "proxy",
        "auto_detect_interface": true
    },
    "inbounds": [
        {
            "tag": "tun-in",
            "type": "tun",
            "interface_name": "sing-box",
            "inet4_address": "172.19.0.1/30",
            "inet6_address": "fdfe:dcba:9876::1/126",
            "inet4_route_address": "198.18.0.0/16",
            "inet6_route_address": "fc00::/18",
            "mtu": 1420,
            "auto_route": true,
            "stack": "mixed",
            "sniff": true
        }
    ],
    "outbounds": [
        {
            "tag": "proxy",
            // 略
        },
        {
            "tag": "direct-out",
            "type": "direct"
        },
        {
            "tag": "dns-out",
            "type": "dns"
        }
    ]
}
```

## 附
如果你确保上述操作正确的话（特别注意关闭系统代理），如果打开 YouTube 的视频一直无法加载，可能浏览器依然首先尝试了一般的 TLS 连接，具体无法确定原因，解决方法暂无（多刷新几次/重启浏览器/重启电脑 可能在某次就进行 QUIC 连接了），不过一旦浏览器开始 QUIC 连接后，后续非常稳定

# 优点和缺点？
一个必然的事实是，由于你直连了 YouTube 的 CDN，对有区域版权限制的媒体自然是无法访问的

如果你是自建节点，单点大流量的特征是无论如何无法避免的，将 YouTube 视频流量直连，可减少经过代理的流量，降低风险

而如果你节点速度不佳，直连 YouTube 可能大幅提升观影体验（如果 YouTube 给你分配了 HK 的 CDN 的话，甚至能跑到 20w+，不过目前分配机制较迷，有时也会分配荷兰或者洛杉矶等地的 CDN） 