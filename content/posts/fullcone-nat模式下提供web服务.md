---
title: fullcone-nat 模式下提供 web 服务
date: 2023-10-15 12:33:46
tags:
  - Network
  - NAT
  - Fullcone
categories: 
  - 技巧分享
url: /posts/web-with-fullcone
---
老实说，3202 年要提供使用家里云对外提供 web服务，直接用 ipv6 就好了

事实上，我本来也是如此，但毕竟还是要和某些只有 ipv4 的“小可爱”做朋友的，在受够了教别人如何开启 ipv6 后，我决定寻求无公网  ipv4 支持家里云双栈访问的方法

# 探索和理论
### 方案选择
我家里是移动宽带，申请公网 ipv4 是不用想了

那么，一种简单的想法是通过一台双栈的主机以实现 `4to6` 的转换

但鉴于国内云服务器的“小水管”，不足以满足我 NAS 高带宽的需求，并且还有备案的麻烦事，很快被我 pass 了

然后是国外主机，虽然支持高带宽的传输，但流量要从国内->国外->国外的转发，延迟爆炸，实在不甚优雅，同时也要每月付出一笔不菲的费用
~~还有可能由于某些原因被阻断~~

我也考虑过套 cloudflare 回源家里云，不得不说这是一场灾难，打开 web 基本都要十几秒，速度也是十分堪忧，~~非常符合我对流量经过国外的想象~~
并且 cloudflare 作为 CDN 服务器能解密中间所有流量，即使 cloudflare 声称保护互联网隐私，但对于家里云里的私密信息，果然还是放心不下

难道真的没有完美的方案了吗？

### NAT 穿越
一次机缘巧合，我注意到了家里的移动宽带使用的是 `fullcone`，也就是所谓的 `nat1`
> 至于什么是 nat 类型，可自行搜索，此处不赘述

这给我极大的希望，因为 nat1 几乎和公网 ipv4 无异，很快我找到了 nat 打洞的方案：[natmap](https://github.com/heiher/natmap)

不过，在这个项目的 wiki 里，我只找到了一种非常不优雅的[解决方法](https://github.com/heiher/natmap/wiki/web)，它依然依赖 cloudflare的反代
~~如果一定要经过 cf，直接用 cf tunnel 不就得了，还打什么洞啊~~

当然，它依然十分有用，`natmap` 已经实现 NAT 穿越，上面的套 cloudflare 方案不过为了解决动态端口的问题，只是这点令我不甚满意

### 动态端口问题
我们都知道，动态 IP 可以靠 DDNS 解决，但是端口却不在 DNS 的范畴
> 虽然 HTTPS 记录已经出现 port 键，但几乎所有的客户端都不支持

那么，使用 302 重定向呢？

当这个想法从脑海中闪现时，我立刻兴奋起来：是的，302 重定向可以让将访问重定向并带上端口号

以下是网络架构的思路：
1. natmap 实现 NAT 穿越，并获取 NAT 出口的 IP 和端口
2. IP 使用 DDNS 绑定域名`v4.homecloud.com`，端口上报到 302 服务器`302.homecloud.com`
3. 302 服务器将每个访问`https://302.homecloud.com/path?search=` 302 跳转到 `https://v4.homecloud.com:${port}/path?search=`

注意到 302 服务器仅完成 302 跳转，所以无论使用国内云服务器（低延迟小带宽），还是国外服务器（高延迟），都不影响访问体验！

当然，我决定更进一步，302 这点事完全可以交给 cloudflare workers

# 实际操作
> 假设家里云在 2023 端口提供 web 服务

### 配置 cloudflare workers
workers 本来并没有存储功能，好在 cloudflare 给了另一项功能：KV空间

1. 创建一个 workers 项目 `homecloud` 和一个 KV 空间 `hc`
2. 将 `hc` 绑定至 `homecloud` 并命名为 `nat_port`
3. 绑定域名 `302.homecloud.com` 至 `homecloud`
4. 然后在 `homecloud` 内编辑以下代码：
```javascript cloudflare-workers
export default {
    async fetch(request, env) {
        const { pathname, search } = new URL(request.url);
        // 提交端口到 KV 空间
        if (pathname === "/port") {
            if (request.method === "POST") {
                const port = await request.json();
                await env.nat_port.put("port", port);
                return new Response(port, {
                    headers: { "Content-Type": "application/json" },
                });
            } else if (request.method === "GET") {
                const value = await env.nat_port.get("port");
                return new Response(value);
            }
            else {
                return new Response("error! ");
            }
        }
        // 如果访问者支持 ipv6，则跳转至 ipv6 域名
        const ip = String(request.headers.get('CF-Connecting-IP'));
        if (ip.includes(":")) {
            const v6 = "https://v6.homecloud.com:2023";
            const statusCode = 302;
            const destinationURL = `${v6}${pathname}${search}`;
            return Response.redirect(destinationURL, statusCode);
        }
        // 否则，跳转至 nat 域名
        const port = await env.nat_port.get("port");
        const v4 = "https://v4.homecloud.com:";
        const statusCode = 302;
        const destinationURL = `${v4}${port}${pathname}${search}`;
        return Response.redirect(destinationURL, statusCode);
    }
}
```
> 注意将域名和端口替换成你自己的

### 配置 natmap
1. 新建脚本 `/usr/bin/nat-port`，并填入以下内容
2. 在 openwrt 中运行命令：`natmap -d -i pppoe-wan -s stunserver.stunprotocol.org -h qq.com -b 2023 -e /usr/bin/nat-port`
```bash nat-port
#!/bin/sh

# 端口上报至 cloudflare workers
ADDR=${1}
PORT=${2}
WORKERS="302.homecloud.com"

curl -X "POST" "https://$WORKERS/port" \
     -H "Content-Type: application/json; charset=utf-8" \
     -d "$PORT"

# DDNS
CFKEY="your cloudflare API key"
CFUSER=""

CFZONE_NAME="homecloud.com"
# run
# curl -s -X GET "https://api.cloudflare.com/client/v4/zones?name=$CFZONE_NAME" -H "X-Auth-Email: $CFUSER" -H "X-Auth-Key: $CFKEY" -H "Content-Type: application/json"
# to get this ID
CFZONE_ID=""

CFRECORD_NAME="v4.homecloud.com"
# run
# curl -s -X GET "https://api.cloudflare.com/client/v4/zones/$CFZONE_ID/dns_records?name=$CFRECORD_NAME" -H "X-Auth-Email: $CFUSER" -H "X-Auth-Key: $CFKEY" -H "Content-Type: application/json"
# to get this ID
CFRECORD_ID=""
CFRECORD_TYPE=A
CFTTL=120

RESPONSE=$(curl -s -X PUT "https://api.cloudflare.com/client/v4/zones/$CFZONE_ID/dns_records/$CFRECORD_ID" \
  -H "X-Auth-Email: $CFUSER" \
  -H "X-Auth-Key: $CFKEY" \
  -H "Content-Type: application/json" \
  --data "{\"id\":\"$CFZONE_ID\",\"type\":\"$CFRECORD_TYPE\",\"name\":\"$CFRECORD_NAME\",\"content\":\"$ADDR\", \"ttl\":$CFTTL}")
```
> 注意填入必要的信息