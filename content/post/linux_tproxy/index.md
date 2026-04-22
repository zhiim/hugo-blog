+++

title = "Linux 透明代理（TProxy）配置"
date = 2026-04-22T09:54:55+08:00
slug = "linux-tproxy"
description = "本文介绍了使用 sing-box 或者 mihomo 代理客户端时，如何基于 iptables 规则实现透明代理"
tags = ["Linux", "工具"]
categories = ["Note"]
image = ""

+++

现代代理客户端（例如 sing-box 和 mihomo）已经支持非常精细而成熟的分流规则，我们完全可以使用全局透明代理实现最优的代理体验，而不用担心直连域名误走代理出站。

在 Linux 平台全局透明代理主要有基于内核防火墙的 iptables 模式和基于虚拟网卡的 TUN 模式。TUN 模式将原始到 IP 数据包拦截到虚拟网卡，代理软件需要运行一套完整的 TCP/IP 协议栈（如 gVisor）模拟 TCP 握手和状态机，开销极大。基于 iptables 的透明代理只需要通过内核态的 iptables 规则拦截内核处理好的 TCP 数据流并转交给代理软件，更加干净高效，此外 iptables 可以实现非常精细的拦截，例如放行 Tailscales 和 libvirt 虚拟机的流量。

鉴于此，本文将介绍基于 iptables 实现的透明代理，涵盖针对 sing-box 和 mihomo 的 DNS 处理机制不同处理，以及放行 Tailscales 和 libvirt 流量的精细化规则。

正因如此，本文将介绍基于 iptables 的透明代理，针对 sing-box 与 mihomo 不同的 DNS 处理设置了对应的规则，并添加了放行 Tailscale 与 libvirt 虚拟机流量的配置。

## iptables 简介

iptables 是 Linux 内核 Netfilter 防火墙的用户态配置工具，可以通过一套规则化的表达配置 Linux 内核防火墙的工作链路。iptables 规则由 **Tables-Chains-Rules-Target** 构成

- **Tables（表）**：将规则按照功能放在不同的表中，默认包含 Filter（决定哪些数据包可以出入）、NAT（修改数据包的目标和源 IP 以及端口号）、Mangle（修改数据表的 IP 报头信息）、Raw（让特定流量不被内核追踪）
- **Chains（链）**：数据包在系统中会依次经过不同的链，默认有 PREROUTING、INPUT、FORWARD、OUTPUT 和 POSTROUTING
- **Rules（规则）**：匹配数据包的条件，例如源 IP 地址为 127.0.0.0 的 udp 数据包可以被匹配
- **Targets（动作）**：数据包被匹配后执行的动作，例如 ACCEPT、DROP、REDIRECT

{{<figure src="iptables.webp" title="Netfilter 数据处理流程图">}}

图中画出了数据包从入站到出站在 Netfilter 中依次经过的链和表，我们可以由三种场景窥探数据流向：

- 路由器中来自局域网设备访问互联网的流量：`POSTROUTINGRE -> FORWARD -> POSTROUTING`
- 来自其他设备的入站流量：``

