---
layout: post
date: 2017-06-01 10:25:00
title: iOS VPN 开发
category: 技术
keywords: iOS
description: iOS 上进行 VPN 连接
---

本文只讨论技术，而不做它用，坚决遵守国家的政策 ： )

# 通信原理

## 科学上网为什么可以成功

### 原本的上网姿势

如图，在 GFW 出现之前，我们访问各种网站是这样的：

1.  用户的请求通过互联网发送到服务提供方
2.  服务提供方直接将信息反馈给用户

![ss_pre_pc2server](http://7xiym9.com1.z0.glb.clouddn.com/ss_pre_pc2server.png)



### 加入 GFW 之后的上网姿势

![ss_gfw_pc2server](http://7xiym9.com1.z0.glb.clouddn.com/ss_gfw_pc2server.png)

- 加入 GFW 之后，它就像一道墙一样，对信息作筛选和过滤。 request1 / response1 之类的被允许的内容，可以继续通行。
- 而 request2 之类的请求则会被拒绝，收到 Connection Reset 这样的响应内容，而像 谷歌／脸书 一类的网站将永不能被 GFW 内的 PC 访问。

### SSH Tunnel

因为SSH本身基于RSA加密技术，所以GFW就无法对数据传输过程加密的数据进行分析，从而避免被重置链接、阻断、屏蔽等问题。

由于在创建SSH隧道的过程中有较为明显的特性，所以GFW还是可以通过分析连接的特性进行干扰。

SOCKS5 协议作为一个同时支持 TCP 和 UDP 的应用层协议（RFC 只有短短的 7 页），因为其简单易用的特性而被 shadowsocks 青睐。我们先从 SOCKS5 协议入手，一点一点剖析 shadowsocks。

SOCKS5 协议只负责建立连接，在完成握手阶段和建立连接之后，SOCKS5 服务器就只做简单的转发了


### 改造 SSH 后的 Shadowsocks

![vpn_shadowshocks](http://7xiym9.com1.z0.glb.clouddn.com/vpn_ss_socks5.png)



## Shadowsocks 分析

### 基础知识

#### TCP/IP 协议栈

TCP/IP网络协议栈分为应用层（Application）、传输层（Transport）、网络层（Network）和链路层（Link）四层

(按TCP/IP参考模型划分) 
应用层 FTP SMTP HTTP ... tcp/ip协议栈
传输层 TCP UDP 
网络层 IP ICMP ARP 
链路层 以太网 令牌环 FDDI ... 
包含了一系列构成互联网基础的网络协议。
这些协议最早发源于美国国防部的DARPA互联网项目。
TCP/IP字面上代表了两个协议:TCP传输控制协议和IP互联网协议。 

#### Socket5协议 

首先介绍一下socks5协议： SOCKS协议位于传输层(TCP/UDP等)与应用层之间，其工作流程为：

1. client向proxy发出请求信息，用以协商传输方式
2. proxy作出应答
3. client接到应答后向proxy发送目的主机（destination server)的ip和port
4. proxy评估该目的主机地址，返回自身IP和port，此时C/P连接建立。
5. proxy与dst server连接
6. proxy将client发出的信息传到server，将server返回的信息转发到client。代理完成

##### 握手阶段

##### 建立连接

##### 传输阶段


### 协议与结构


### 加密代理协议的简单对比

目前我们常用的加密代理有协议有 HTTPS，SOCKS5-TLS 和 shadowsocks,此文从各个角度简单分析各个协议的优劣，以帮助各位选择合适的协议。
先简单说些背景知识，以上协议都是基于 TCP 的代理协议，代理协议（Proxy Procotol）与 VPN 不同，仅可被用于通过代理服务器转发 TCP 连接（shadowsocks 支持代理 UDP），而 VPN 可被用于 IP 层上的所有协议，如 TCP、UDP、ICMP 等。所以在使用代理时，ping 等 ICMP 应用是不可以被代理的。

#### TLS

TLS 又名 SSL，是针对数据流的安全传输协议。简单来说，一个 TCP 链接，把其中原本要传输的数据，按照 TLS 协议去进行加密传输，那么即可保证其中传输的数据安全。

明文的 HTTP 套上一层 TLS，也就变成了 HTTPS，SOCKS5 套上 TLS，就变成了 SOCKS5-TLS。TLS 协议是整个互联网安全的基石，几乎所有需要安全传输的地方都使用了 TLS，如银行、政府等等。

#### shadowsocks 



# iOS 客户端实现 

## Proxy Server

## DNS Client

## TUN Interface

用 TUN 处理请求会有一些问题，最大的问题是，由 TUN Interface 处理的流量，DOMAIN 相关的 Rule 会无效，除非使用了 force-remote-dns 选项。

这是由 TCP/IP 协议的特性所决定的，App 会先发出一个 DNS question，获取要连接的服务器的 IP 地址，然后直接向这个 IP 地址发起连接

## local server

Now we can start a HTTP/SOCKS5 proxy server locally.

```
let server = GCDHTTPProxyServer(address: IPv4Address(fromString: "127.0.0.1"), port: Port(port: 9090)
try! server.start()
```

## iOS Extension

### Extension 调试  

# 参考

[TCP/IP协议栈与数据包封装](https://akaedu.github.io/book/ch36s01.html)

[Shadowsocks 源码解释](http://yveschan.github.io/blog/shadowsocks-analysis/)

[Shadowsocks 源码分析——协议与结构](https://loggerhead.me/posts/shadowsocks-yuan-ma-fen-xi-xie-yi-yu-jie-gou.html)

[各种加密代理协议的简单对比--surge作者](https://medium.com/@Blankwonder/%E5%90%84%E7%A7%8D%E5%8A%A0%E5%AF%86%E4%BB%A3%E7%90%86%E5%8D%8F%E8%AE%AE%E7%9A%84%E7%AE%80%E5%8D%95%E5%AF%B9%E6%AF%94-1ed52bf7a803)

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


