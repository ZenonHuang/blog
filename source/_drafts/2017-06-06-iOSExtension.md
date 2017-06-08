---
layout: post
date: 2017-06-01 10:25:00
title: iOS VPN 开发
category: 技术
keywords: iOS
description: iOS 上进行 VPN 连接
---

# 代理通信原理

## Shadowsocks 源码分析——协议与结构

![vpn_shadowshocks](http://7xiym9.com1.z0.glb.clouddn.com/vpn_ss_socks5.png)

### 协议对比

目前我们常用的加密代理有协议有 HTTPS，SOCKS5-TLS 和 shadowsocks,此文从各个角度简单分析各个协议的优劣，以帮助各位选择合适的协议。
先简单说些背景知识，以上协议都是基于 TCP 的代理协议，代理协议（Proxy Procotol）与 VPN 不同，仅可被用于通过代理服务器转发 TCP 连接（shadowsocks 支持代理 UDP），而 VPN 可被用于 IP 层上的所有协议，如 TCP、UDP、ICMP 等。所以在使用代理时，ping 等 ICMP 应用是不可以被代理的。

# 实现

## ****TCP/IP 协议栈

TCP/IP网络协议栈分为应用层（Application）、传输层（Transport）、网络层（Network）和链路层（Link）四层

(按TCP/IP参考模型划分) 
应用层 FTP SMTP HTTP ... tcp/ip协议栈
传输层 TCP UDP 
网络层 IP ICMP ARP 
链路层 以太网 令牌环 FDDI ... 
包含了一系列构成互联网基础的网络协议。
这些协议最早发源于美国国防部的DARPA互联网项目。
TCP/IP字面上代表了两个协议:TCP传输控制协议和IP互联网协议。 

## local server

Now we can start a HTTP/SOCKS5 proxy server locally.

```
let server = GCDHTTPProxyServer(address: IPv4Address(fromString: "127.0.0.1"), port: Port(port: 9090)
try! server.start()
```

# iOS Extension

# Extension 调试  

# 参考

[TCP/IP协议栈与数据包封装](https://akaedu.github.io/book/ch36s01.html)

[Shadowsocks 源码解释](http://yveschan.github.io/blog/shadowsocks-analysis/)

[Shadowsocks 源码分析——协议与结构](https://loggerhead.me/posts/shadowsocks-yuan-ma-fen-xi-xie-yi-yu-jie-gou.html)

[各种加密代理协议的简单对比](https://medium.com/@Blankwonder/%E5%90%84%E7%A7%8D%E5%8A%A0%E5%AF%86%E4%BB%A3%E7%90%86%E5%8D%8F%E8%AE%AE%E7%9A%84%E7%AE%80%E5%8D%95%E5%AF%B9%E6%AF%94-1ed52bf7a803)

[TCP/IP、Http、Socket的区别](http://lib.csdn.net/article/computernetworks/20534)

[socket,socket5](http://www.jianshu.com/p/515c6d567d93)

[从SSL安全传输到iOS证书安全体系](http://blog.csdn.net/fanyiyao980404514/article/details/44859783)

[基于 iOS 实现 socks5 代理](http://www.jianshu.com/p/08fc1f68120a)

[为什么不该用ssl翻墙](https://gist.github.com/clowwindy/5947691)

[写给非专业人士看的 Shadowsocks 简介](http://vc2tea.com/whats-shadowsocks/)

[ss翻墙原理](https://tumutanzi.com/archives/13005)

[Shadowsocksr](https://github.com/shadowsocksr)

[Shadowsocks](https://zh.wikipedia.org/wiki/Shadowsocks)

[iOS开发之VPN协议（理论）](http://blog.csdn.net/dengshuai_super/article/details/51872474)

[NetworkExtension之NEVPNManager开发笔记](https://blog.6ag.cn/1714.html)

[VPN应用](http://www.jianshu.com/p/89265b0ce635)

[extension 介绍](http://www.jianshu.com/p/bbc6a95d9c54)

[extension 开发](http://www.jianshu.com/p/c4a27bfc9917)


