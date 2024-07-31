---
slug: a-trouble-of-tproxy
title: 折腾 tproxy 下的一点踩坑
date: 2023-07-28 14:15:06
tags: 
  - Network
  - Tproxy
categories: 
  - 记录
summary: from end user
---
# 起因
其实早在三天前我就打算写出第一篇博客了，但是由于我国互联网的特殊性，导致安装 hexo 出现了“一点点”小意外。虽然只需要设置一下命令行代理就能搞定

`set http_proxy=http://ip:port`

但是这件事给我了足够的动力去折腾透明网关。~~其实早就想搞的，拖延症+懒，就。。。~~
# 技术介绍
tproxy，意为透明代理/透明网关，通过在路由器上运行代理软件并劫持局域网内所有流量便可实现局域网设备“无感”翻墙

它相对与传统的 http 代理或者 sock 代理有以下优点：
1. 使不支持设置代理的设备翻墙，例如 apple TV，switch等
2. 使不遵守系统代理的软件走代理
3. 对代理设备透明，无需对局域网中设备做任何调整

* 如果仅是第二条的话，利用 tun 模式即可实现，但这就不在本文的范围内了，请自行查阅相关资料
* 目前有许多十分完善的代理插件，如 [openclash](https://github.com/vernesong/OpenClash) [helloworld](https://github.com/jerrykuku/luci-app-vssr) [passwall](https://github.com/xiaorouji/openwrt-passwall) 等，如果你**不**熟悉Linux操作/善用Google/懂得[提问的智慧](https://github.com/ryanhanwu/How-To-Ask-Questions-The-Smart-Way/blob/main/README-zh_CN.md)，请直接使用插件
# 具体操作
本文主要参考 [这个](https://xtls.github.io/document/level-2/transparent_proxy/transparent_proxy.html)，本文并不打算复述该文章的内容，仅仅对该文没讲到的地方做出补充。特别的，我并没有将网关纳入透明代理中，因为我觉得没必要且相当麻烦（需要排除xray的出站流量）

总之，我最后的iptables配置如下：
```bash
#!/bin/bash
ip rule add fwmark 1 table 100
ip route add local 0.0.0.0/0 dev lo table 100
iptables -t mangle -N XRAY
iptables -t mangle -A XRAY -d 10.0.0.0/8 -j RETURN
iptables -t mangle -A XRAY -d 100.64.0.0/10 -j RETURN
iptables -t mangle -A XRAY -d 127.0.0.0/8 -j RETURN
iptables -t mangle -A XRAY -d 169.254.0.0/16 -j RETURN
iptables -t mangle -A XRAY -d 172.16.0.0/12 -j RETURN
iptables -t mangle -A XRAY -d 224.0.0.0/3 -j RETURN
iptables -t mangle -A XRAY -d 255.255.255.255/32 -j RETURN
iptables -t mangle -A XRAY ! -s 192.168.0.0/16 -j RETURN
iptables -t mangle -A XRAY -d 192.168.0.0/16 -p tcp -j RETURN
iptables -t mangle -A XRAY -d 192.168.0.0/16 -p udp ! --dport 53 -j RETURN
iptables -t mangle -A XRAY -p tcp -j TPROXY --on-port 12345 --tproxy-mark 1
iptables -t mangle -A XRAY -p udp -j TPROXY --on-port 12345 --tproxy-mark 1
iptables -t mangle -A PREROUTING -j XRAY

sleep 10

ipv6=$(ip -6 route show dev lo)
ipv6_CIDR=${ipv6: 12: 24}
ip -6 rule add fwmark 1 table 106
ip -6 route add local ::/0 dev lo table 106
ip6tables -t mangle -N XRAY6
ip6tables -t mangle -A XRAY6 -d ::1/128 -j RETURN
ip6tables -t mangle -A XRAY6 -d fe80::/10 -j RETURN
ip6tables -t mangle -A XRAY6 ! -s ${ipv6_CIDR} -j RETURN
ip6tables -t mangle -A XRAY6 -d ${ipv6_CIDR} -p tcp -j RETURN
ip6tables -t mangle -A XRAY6 -d ${ipv6_CIDR} -p udp ! --dport 53 -j RETURN
ip6tables -t mangle -A XRAY6 -p udp -j TPROXY --on-port 12345 --tproxy-mark 1
ip6tables -t mangle -A XRAY6 -p tcp -j TPROXY --on-port 12345 --tproxy-mark 1
ip6tables -t mangle -A PREROUTING -j XRAY6
```
## 与参考文章相比，我做出哪些改变？
事实上，我主要增加了 ipv6 的部分，即便是 [这篇](https://xtls.github.io/document/level-2/tproxy_ipv4_and_ipv6.html)，也仅仅针对使用 nat6 的网络，我相信使用 ipv6 肯定是图它全球单播的优点，不然我觉得直接关闭 ipv6 是更省事的做法（使用 fd00::/8 是 unacceptable！！！）
但是针对我国 ISP 使用的动态 ipv6-pd 的问题，每次重新拨号都会使前缀变化，网段肯定不能写死在配置里，因此随系统启动时要能主动获取当前的网段：
1. 运行`ip -6 route show dev lo`获取本机的 ipv6 网段信息
2. `ipv6_CIDR=${ipv6: 12: 24}`截取网段信息
* 注意`ip -6 route show dev lo`时可能会返回多个 ipv6 地址，注意区分哪个是 pd 前缀
* `sleep 10`是为了等待 pppoe 拨号，拨号后才可能下发前缀


# 踩坑
实际上，调 iptables 的配置，不能说是一番风顺吧，只能说是举步维艰
一些很快被发现的小问题就不说了。~~几次写错 iptables 的规则导致 ssh 都访问不到路由器，只能断电重启~~

一开始，我对 DNS 的处理是这样的
```bash
iptables -t mangle -A XRAY -d 192.168.0.0/16 -p tcp ! --dport 53 -j RETURN
iptables -t mangle -A XRAY -d 192.168.0.0/16 -p udp ! --dport 53 -j RETURN
```
而我的 xray 路由中却是只有
```json
{
    "type": "field",
    "inboundTag": [
        "all-in"
    ],
    "port": 53,
    "network": "udp",
    "outboundTag": "dns-out"
}
```
这就导致了局域网请求 TCP 53 时，走了`direct`，这条DNS出了 xray 又进了系统 DNS 查询，系统就先对自己的 TCP 53 端口进行了 DNS 查询，于是这条 DNS 又进了 xray，在这种递归之下，很快，xray 日志里出现了大量这样的日志
```
2023/07/26 14:12:10 192.168.2.2:59754 accepted udp:192.168.2.1:53 [all-in -> dns-out]   //udp 请求就不会，因为它正常的路由到了 dns-out
2023/07/26 14:12:10 192.168.2.2:51312 accepted tcp:192.168.2.1:53 [all-in -> direct]    //这条开始引起了下面反复请求
2023/07/26 14:12:10 192.168.2.1:34072 accepted tcp:192.168.2.1:53 [all-in -> direct]    //这样的请求在实际日志中持续了上百条，这里只是节选
2023/07/26 14:12:10 192.168.2.1:34080 accepted tcp:192.168.2.1:53 [all-in -> direct]
2023/07/26 14:12:10 192.168.2.1:34094 accepted tcp:192.168.2.1:53 [all-in -> direct]
```
在这之后不久，软路由系统就卡死，网络完全崩了

~~当时看了半天 iptables 的原理，脑子完全僵住了，还以为是 xray 的问题，还去提了个 issue~~

# 写在最后
折腾这玩意最好先在其他设备调好再转到主路由上去，~~否则可能引起家庭矛盾~~，而且这一套规则可能之后还会再调，目前还发现了不少小问题

emmmmm，以后再写吧！