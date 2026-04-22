+++

title = "Linux 透明代理（TPROXY）配置"
date = 2026-04-22T09:54:55+08:00
slug = "linux-tproxy"
description = "本文介绍了如何利用 iptables 的 TPROXY 模块，为 sing-box 和 mihomo 等代理客户端配置干净高效的 Linux 全局透明代理"
tags = ["Linux", "工具"]
categories = ["Notes"]
image = ""

+++

现代代理客户端（例如 sing-box 和 mihomo）已经支持非常精细而成熟的分流规则，我们完全可以使用全局透明代理实现最优的代理体验，而不用担心直连域名误走代理出站。

在 Linux 平台全局透明代理主要有基于内核防火墙的 iptables 模式和基于虚拟网卡的 TUN 模式。TUN 模式将原始的 IP 数据包拦截到虚拟网卡，代理软件需要运行一套完整的 TCP/IP 协议栈（如 gVisor）模拟 TCP 握手和状态机，开销极大。基于 iptables 的透明代理只需要通过内核态的 iptables 规则拦截内核处理好的 TCP 数据流并转交给代理软件，更加干净高效，此外 iptables 可以实现非常精细的拦截，例如放行 Tailscale 的流量。

正因如此，本文将介绍基于 iptables 的透明代理，针对 sing-box 与 mihomo 不同的 DNS 处理设置了对应的规则，并添加了放行 Tailscale 流量的配置。

## iptables 简介

iptables 是 Linux 内核 Netfilter 防火墙的用户态配置工具，可以通过一套规则化的表达配置 Linux 内核防火墙的工作链路。iptables 规则由 **Tables-Chains-Rules-Target** 构成

- **Tables（表）**：将规则按照功能放在不同的表中，默认包含 Filter（决定哪些数据包可以出入）、NAT（修改数据包的目标和源 IP 以及端口号）、Mangle（修改数据包的 IP 报头信息）、Raw（让特定流量不被内核追踪）
- **Chains（链）**：数据包在系统中会依次经过不同的链，默认有 PREROUTING、INPUT、FORWARD、OUTPUT 和 POSTROUTING
- **Rules（规则）**：匹配数据包的条件，例如源 IP 地址为 127.0.0.0 的 udp 数据包可以被匹配
- **Targets（动作）**：数据包被匹配后执行的动作，例如 ACCEPT、DROP、REDIRECT

{{<figure src="iptables.webp" title="Netfilter 数据处理流程图">}}

图中画出了数据包从入站到出站在 Netfilter 中依次经过的链和表，我们可以由三种场景窥探数据流向：

- 路由器中来自局域网设备访问互联网的流量，它们的目的 IP 是局域网外，所以会被路由器转发，流向：`PREROUTING -> FORWARD -> POSTROUTING`
- 来自其他设备的入站流量，由 INPUT 链中的 filter 表规则决定是否可以入站（例如 SSH 登录），流向：`PREROUTING -> INPUT -> 本机`
- 本机产生的发往外站的流量，由 OUTPUT 链中的 filter 表规则决定是否放行，流向：`本机 -> OUTPUT -> POSTROUTING`

标准的 iptables 命令通常包含了要操作的链、匹配条件以及执行的动作。例如，最基础的服务放行配置：

```shell
# 允许本地回环接口 (loopback) 的所有流量
iptables -A INPUT -i lo -j ACCEPT

# 允许目标端口为 80 (HTTP) 和 22 (SSH) 的 TCP 流量入站
iptables -A INPUT -p tcp --dport 80 -j ACCEPT
iptables -A INPUT -p tcp --dport 22 -j ACCEPT

# 将特定 IP 来源的流量直接丢弃且不产生回应
iptables -A INPUT -s 192.168.1.100 -j DROP
```

## 基于 iptables 实现透明代理

本节我们基于 iptables 实现透明代理：通过 iptables 规则将系统中的网络流量劫持并转发到代理客户端，然后经过代理客户端打包后发出。假设代理客户端（可以是 sing-box 或 mihomo）的 tproxy 入站端口设置为 9898。

### 劫持流量到代理客户端

Linux 透明代理可以使用 TPROXY 模块，它能完美处理 TCP 与 UDP 流量并保留真实源 IP，但是 TPROXY 模块只能挂载在 mangle 表的 PREROUTING 链。

局域网设备访问互联网的流量链路为 `PREROUTING -> FORWARD -> POSTROUTING`，只需要将 PREROUTING 链上的流量劫持到代理客户端的 tproxy 入站端口，就能实现流量被代理客户端转发。

设备自身产生的访问互联网的流量链路为 `本机 -> OUTPUT -> POSTROUTING`，并不会经过 PREROUTING 链。为了实现本机流量也走代理，可以将 OUTPUT 链的流量由回环设备 lo 发出，这样会自动被 lo 再次接受，流量就像外来包一样重新入站走 `PREROUTING -> INPUT -> 本机`，下一站就是 PREROUTING 链，之后被劫持到代理客户端的 tproxy 端口。

所以我们的透明代理分为两个步骤实现：

1. 将 OUTPUT 链的流量由本地回环设备 lo 发出，让流量自动出现在 PREROUTING 链；
1. 劫持 PREROUTING 链并转发到代理客户端。

#### 将 OUTPUT 链的流量发往 lo 设备

在处理 OUTPUT 链流量时，需要放行由代理客户端发出的出站流量，否则流量会再次转发出现回环。为了解决这个问题，我们需要专门创建一个例如 `tproxy` 用户用于运行代理客户端，并且在 iptables 规则中放行 `tproxy` 用于发出的流量。

```shell
# 创建 tproxy 用户
sudo useradd -r -M -s /usr/sbin/nologin tproxy
# 调整权限，让代理客户端的配置文件属于 tproxy 用户
sudo chown -R tproxy:tproxy /usr/local/share/sing-box
```

创建 systemd 服务用于运行代理客户端

```toml
[Unit]
Description=sing-box Service
After=network-online.target

[Service]
ExecStartPre=+/usr/sbin/setcap 'cap_net_admin,cap_net_bind_service,cap_net_raw,cap_sys_ptrace,cap_dac_read_search+ep' /usr/local/bin/sing-box
ExecStart=/usr/local/bin/sing-box run -c /usr/local/share/sing-box/config.json
WorkingDirectory=/usr/local/share/sing-box
RestartSec=3
User=tproxy
Group=tproxy

[Install]
WantedBy=multi-user.target
```

接下来，新建策略路由，将所有带有标签 1 的流量由 lo 设备发出。然后新建一个 TRANS_PROXY_MASK 链，将所有经过 TRANS_PROXY 链的流量打上标签 1，这样这些流量会自动由 lo 设备发出。最后只需要将不属于 `tproxy` 用户的出站流量由 OUTPUT 链劫持到 TRANS_PROXY_MASK 链。

```shell
# mask 被标记为 1 的数据包查询编号为 100 的路由表
ip rule add fwmark 1 table 100
# 在 100 路由表中将来源与任何 IP 的数据包标记为本地流量，并发由 lo 设备发出
ip route add local 0.0.0.0/0 dev lo table 100

# 创建 TRANS_PROXY_MASK 链
iptables -t mangle -N TRANS_PROXY_MASK
# 为 TRANS_PROXY_MASK 链中的流量打上 mask 1 实现劫持到 lo 设备
iptables -t mangle -A TRANS_PROXY_MASK -p tcp -j MARK --set-mark 1
iptables -t mangle -A TRANS_PROXY_MASK -p udp -j MARK --set-mark 1

# 劫持非 tproxy 用户的 OUTPUT 流量到 TRANS_PROXY_MASK 链
iptables -t mangle -A OUTPUT -m owner ! --uid-owner tproxy -j TRANS_PROXY_MASK
```

#### 劫持 PREROUTING 链流量到代理客户端

新建一个 TRANS_PROXY 链，将经过 PREROUTING 链的流量拦截到 TRANS_PROXY 链，然后让经过 TRANS_PROXY 的所有 tcp 和 udp 流量发往透明代理的监听端口。

```shell
# 创建 TRANS_PROXY 链
iptables -t mangle -N TRANS_PROXY

# 将 tcp 和 udp 流量转发到 TRANS_PROXY，并且打上 mask 1 作为本地流量
iptables -t mangle -A TRANS_PROXY -p udp -j TPROXY --on-ip 127.0.0.1 --on-port 9898 --tproxy-mark 1
iptables -t mangle -A TRANS_PROXY -p tcp -j TPROXY --on-ip 127.0.0.1 --on-port 9898 --tproxy-mark 1

# 劫持 PREROUTING
iptables -t mangle -A PREROUTING -j TRANS_PROXY
```

这里将流量发向 TRANS_PROXY 同样打上了标签 1，这是因为 PREROUTING 的下一步是根据目的 IP 判断走 FORWARD 链发往外部，还是走 INPUT 发往本地服务。打上标签 1 后会被策略路由看作 local 流量走 INPUT 发往本地的 9898 端口被代理客户端处理。

### 精细化放行

在上述路由转发规则的基础上，我们还需要设置放行规则，防止错误拦截。例如目的 IP 为同局域网设备的流量、Tailscale 的出入站流量不应该被拦截到代理客户端。

为了放行目的 IP 为同局域网设备的流量，可以在 TRANS_PROXY_MASK 和 TRANS_PROXY 链的最前端（规则是由前往后匹配的，所以这些规则需要最先添加）添加放行的规则。

```shell
iptables -t mangle -A TRANS_PROXY_MASK -d 10.0.0.0/8 -j RETURN
iptables -t mangle -A TRANS_PROXY_MASK -d 172.16.0.0/12 -j RETURN
iptables -t mangle -A TRANS_PROXY_MASK -d 192.168.0.0/16 -j RETURN
iptables -t mangle -A TRANS_PROXY_MASK -d 127.0.0.0/8 -j RETURN
iptables -t mangle -A TRANS_PROXY_MASK -d 224.0.0.0/4 -j RETURN
iptables -t mangle -A TRANS_PROXY_MASK -d 255.255.255.255/32 -j RETURN
```

```shell
iptables -t mangle -A TRANS_PROXY -d 10.0.0.0/8 -j RETURN
iptables -t mangle -A TRANS_PROXY -d 172.16.0.0/12 -j RETURN
iptables -t mangle -A TRANS_PROXY -d 192.168.0.0/16 -j RETURN
iptables -t mangle -A TRANS_PROXY -d 127.0.0.0/8 -j RETURN
iptables -t mangle -A TRANS_PROXY -d 224.0.0.0/4 -j RETURN
iptables -t mangle -A TRANS_PROXY -d 255.255.255.255/32 -j RETURN
```

Tailscale 组网设备的 IP 段为 `100.64.0.0/10`，并且 Tailscale 发出的流量会被打上标签 `0x40000`，Tailscale 入站的流量会由 `tailscale0` 设备接收。基于这些特征，我们可以添加规则不拦截 Tailscale 出入站的流量（同样需要在对应链中优先添加）。

```shell
# 本机发出的目的地址为 tailscale IP 段的流量绕过
iptables -t mangle -I TRANS_PROXY_MASK 1 -d 100.64.0.0/10 -j RETURN
# 本机发出的由 tailscale 产生的流量绕过
iptables -t mangle -I TRANS_PROXY_MASK 2 -m mark --mark 0x40000 -j RETURN
# 从 tailscale0 进入本机的流量绕过
iptables -t mangle -I TRANS_PROXY 1 -i tailscale0 -j RETURN
# 目的地址为 tailscale IP 段的流量绕过
iptables -t mangle -I TRANS_PROXY 2 -d 100.64.0.0/10 -j RETURN
```

### 劫持 DNS 流量

如果想要按照代理客户端的 DNS 规则解析域名，需要将系统的 DNS 请求也劫持到代理客户端。

#### DNS 请求劫持到 sing-box

sing-box 会将 DNS 流量和 TCP/UDP 流量一起处理，只要在 sing-box 的 `route` 字段添加

```json
{
  "action": "hijack-dns",
  "protocol": "dns"
}
```

我们已经拦截了所有出站的 TCP/UDP 流量，所以如果系统设置了公共 DNS 服务器，sing-box 已经可以正常劫持 DNS 请求。如果系统的 DNS 服务器设置为上游网关，由于前面添加了放行目的 IP 为局域网 IP 的规则，需要额外劫持这些 IP 发往 53 端口的 TCP/UDP 流量。在 TRANS_PROXY_MASK 和 TRANS_PROXY 链的最前端添加

```shell
iptables -t mangle -A TRANS_PROXY_MASK -d 10.0.0.0/8 -p udp --dport 53 -j MARK --set-mark 1
iptables -t mangle -A TRANS_PROXY_MASK -d 10.0.0.0/8 -p tcp --dport 53 -j MARK --set-mark 1
iptables -t mangle -A TRANS_PROXY_MASK -d 172.16.0.0/12 -p udp --dport 53 -j MARK --set-mark 1
iptables -t mangle -A TRANS_PROXY_MASK -d 172.16.0.0/12 -p tcp --dport 53 -j MARK --set-mark 1
iptables -t mangle -A TRANS_PROXY_MASK -d 192.168.0.0/16 -p udp --dport 53 -j MARK --set-mark 1
iptables -t mangle -A TRANS_PROXY_MASK -d 192.168.0.0/16 -p tcp --dport 53 -j MARK --set-mark 1
```

```shell
iptables -t mangle -A TRANS_PROXY -d 10.0.0.0/8 -p udp --dport 53 -j TPROXY --on-ip 127.0.0.1 --on-port 9898 --tproxy-mark 1
iptables -t mangle -A TRANS_PROXY -d 10.0.0.0/8 -p tcp --dport 53 -j TPROXY --on-ip 127.0.0.1 --on-port 9898 --tproxy-mark 1
iptables -t mangle -A TRANS_PROXY -d 172.16.0.0/12 -p udp --dport 53 -j TPROXY --on-ip 127.0.0.1 --on-port 9898 --tproxy-mark 1
iptables -t mangle -A TRANS_PROXY -d 172.16.0.0/12 -p tcp --dport 53 -j TPROXY --on-ip 127.0.0.1 --on-port 9898 --tproxy-mark 1
iptables -t mangle -A TRANS_PROXY -d 192.168.0.0/16 -p udp --dport 53 -j TPROXY --on-ip 127.0.0.1 --on-port 9898 --tproxy-mark 1
iptables -t mangle -A TRANS_PROXY -d 192.168.0.0/16 -p tcp --dport 53 -j TPROXY --on-ip 127.0.0.1 --on-port 9898 --tproxy-mark 1
```

#### DNS 请求劫持到 mihomo

mihomo 的 DNS 服务监听在特定端口（例如 1053）

```yaml
dns:
  enable: true
  listen: 0.0.0.0:1053
```

想要使用 mihomo 的 DNS 服务，需要将发往 53 端口的 TCP/UDP 流量劫持为发往 127.0.0.1 的 1053 端口。由于 NAT 表控制了数据包的目的 IP 和端口，需要在 NAT 表上添加 iptables 规则

```shell
# 劫持 dns 流量到 mihomo
iptables -t nat -N MIHOMO_DNS
# 将目标为 53 端口的流量重定向到 mihomo 的 DNS 端口
iptables -t nat -A MIHOMO_DNS -p udp --dport 53 -j REDIRECT --to-ports 1053
iptables -t nat -A MIHOMO_DNS -p tcp --dport 53 -j REDIRECT --to-ports 1053

# 拦截本机的 DNS 请求
iptables -t nat -A OUTPUT -p udp --dport 53 -m owner ! --uid-owner tproxy -j MIHOMO_DNS
iptables -t nat -A OUTPUT -p tcp --dport 53 -m owner ! --uid-owner tproxy -j MIHOMO_DNS
# 拦截局域网设备的 DNS 请求
iptables -t nat -A PREROUTING -p udp --dport 53 -j MIHOMO_DNS
iptables -t nat -A PREROUTING -p tcp --dport 53 -j MIHOMO_DNS
```

### 总结

将上述所有规则放在一次，为了实现透明代理，我们需要一次执行的所有规则为

```shell
sysctl -w net.ipv4.ip_forward=1

# 添加策略路由将标签为 1 的流量看作本地流量并由 lo 本地回环设备发出
ip rule add fwmark 1 table 100
ip route add local 0.0.0.0/0 dev lo table 100

# 添加 TRANS_PROXY_MASK 链
iptables -t mangle -N TRANS_PROXY_MASK
# 放行 Tailscale 入站
iptables -t mangle -I TRANS_PROXY_MASK 1 -d 100.64.0.0/10 -j RETURN
iptables -t mangle -I TRANS_PROXY_MASK 2 -m mark --mark 0x40000 -j RETURN
# 如果劫持 DNS 请求到 sing-box
# iptables -t mangle -A TRANS_PROXY_MASK -d 10.0.0.0/8 -p udp --dport 53 -j MARK --set-mark 1
# iptables -t mangle -A TRANS_PROXY_MASK -d 10.0.0.0/8 -p tcp --dport 53 -j MARK --set-mark 1
# iptables -t mangle -A TRANS_PROXY_MASK -d 172.16.0.0/12 -p udp --dport 53 -j MARK --set-mark 1
# iptables -t mangle -A TRANS_PROXY_MASK -d 172.16.0.0/12 -p tcp --dport 53 -j MARK --set-mark 1
# iptables -t mangle -A TRANS_PROXY_MASK -d 192.168.0.0/16 -p udp --dport 53 -j MARK --set-mark 1
# iptables -t mangle -A TRANS_PROXY_MASK -d 192.168.0.0/16 -p tcp --dport 53 -j MARK --set-mark 1
# 放行目的 IP 为局域网 IP 的流量
iptables -t mangle -A TRANS_PROXY_MASK -d 10.0.0.0/8 -j RETURN
iptables -t mangle -A TRANS_PROXY_MASK -d 172.16.0.0/12 -j RETURN
iptables -t mangle -A TRANS_PROXY_MASK -d 192.168.0.0/16 -j RETURN
iptables -t mangle -A TRANS_PROXY_MASK -d 127.0.0.0/8 -j RETURN
iptables -t mangle -A TRANS_PROXY_MASK -d 224.0.0.0/4 -j RETURN
iptables -t mangle -A TRANS_PROXY_MASK -d 255.255.255.255/32 -j RETURN
# 给所有流经 TRANS_PROXY_MASK 链的流量打上标签 1
iptables -t mangle -A TRANS_PROXY_MASK -p tcp -j MARK --set-mark 1
iptables -t mangle -A TRANS_PROXY_MASK -p udp -j MARK --set-mark 1
# 将所有非 tproxy 用户发出的流量劫持到 TRANS_PROXY_MASK 链
iptables -t mangle -A OUTPUT -m owner ! --uid-owner tproxy -j TRANS_PROXY_MASK

# 添加 TRANS_PROXY 链
iptables -t mangle -N TRANS_PROXY
# 放行 Tailscale 出站
iptables -t mangle -I TRANS_PROXY 1 -i tailscale0 -j RETURN
iptables -t mangle -I TRANS_PROXY 2 -d 100.64.0.0/10 -j RETURN
# 如果劫持 DNS 请求到 sing-box
# iptables -t mangle -A TRANS_PROXY -d 10.0.0.0/8 -p udp --dport 53 -j TPROXY --on-ip 127.0.0.1 --on-port 9898 --tproxy-mark 1
# iptables -t mangle -A TRANS_PROXY -d 10.0.0.0/8 -p tcp --dport 53 -j TPROXY --on-ip 127.0.0.1 --on-port 9898 --tproxy-mark 1
# iptables -t mangle -A TRANS_PROXY -d 172.16.0.0/12 -p udp --dport 53 -j TPROXY --on-ip 127.0.0.1 --on-port 9898 --tproxy-mark 1
# iptables -t mangle -A TRANS_PROXY -d 172.16.0.0/12 -p tcp --dport 53 -j TPROXY --on-ip 127.0.0.1 --on-port 9898 --tproxy-mark 1
# iptables -t mangle -A TRANS_PROXY -d 192.168.0.0/16 -p udp --dport 53 -j TPROXY --on-ip 127.0.0.1 --on-port 9898 --tproxy-mark 1
# iptables -t mangle -A TRANS_PROXY -d 192.168.0.0/16 -p tcp --dport 53 -j TPROXY --on-ip 127.0.0.1 --on-port 9898 --tproxy-mark 1
# 放行目的 IP 为局域网 IP 的流量
iptables -t mangle -A TRANS_PROXY -d 10.0.0.0/8 -j RETURN
iptables -t mangle -A TRANS_PROXY -d 172.16.0.0/12 -j RETURN
iptables -t mangle -A TRANS_PROXY -d 192.168.0.0/16 -j RETURN
iptables -t mangle -A TRANS_PROXY -d 127.0.0.0/8 -j RETURN
iptables -t mangle -A TRANS_PROXY -d 224.0.0.0/4 -j RETURN
iptables -t mangle -A TRANS_PROXY -d 255.255.255.255/32 -j RETURN
# 将所有流经 TRANS_PROXY 链的流量发向代理客户端
iptables -t mangle -A TRANS_PROXY -p udp -j TPROXY --on-ip 127.0.0.1 --on-port 9898 --tproxy-mark 1
iptables -t mangle -A TRANS_PROXY -p tcp -j TPROXY --on-ip 127.0.0.1 --on-port 9898 --tproxy-mark 1
# 将所有流经 PREROUTING 链的流量劫持到 TRANS_PROXY 链
iptables -t mangle -A PREROUTING -j TRANS_PROXY

# 如果劫持 DNS 请求到 mihomo
# iptables -t nat -N MIHOMO_DNS
# iptables -t nat -A MIHOMO_DNS -p udp --dport 53 -j REDIRECT --to-ports 1053
# iptables -t nat -A MIHOMO_DNS -p tcp --dport 53 -j REDIRECT --to-ports 1053
# iptables -t nat -A OUTPUT -p udp --dport 53 -m owner ! --uid-owner tproxy -j MIHOMO_DNS
# iptables -t nat -A OUTPUT -p tcp --dport 53 -m owner ! --uid-owner tproxy -j MIHOMO_DNS
# iptables -t nat -A PREROUTING -p udp --dport 53 -j MIHOMO_DNS
# iptables -t nat -A PREROUTING -p tcp --dport 53 -j MIHOMO_DNS
```

类似地，对于 IPV6 流量可以添加如下规则

```shell
sysctl -w net.ipv6.conf.all.forwarding=1

ip -6 rule add fwmark 1 table 101
ip -6 route add local ::/0 dev lo table 101

ip6tables -t mangle -N TRANS_PROXY_MASK6
ip6tables -t mangle -I TRANS_PROXY_MASK6 1 -d fd7a:115c:a1e0::/48 -j RETURN
ip6tables -t mangle -I TRANS_PROXY_MASK6 2 -m mark --mark 0x40000 -j RETURN
# 如果劫持 DNS 请求到 sing-box
# ip6tables -t mangle -A TRANS_PROXY_MASK6 -d fe80::/10 -p udp --dport 53 -j MARK --set-mark 1
# ip6tables -t mangle -A TRANS_PROXY_MASK6 -d fe80::/10 -p tcp --dport 53 -j MARK --set-mark 1
# ip6tables -t mangle -A TRANS_PROXY_MASK6 -d fc00::/7 -p udp --dport 53 -j MARK --set-mark 1
# ip6tables -t mangle -A TRANS_PROXY_MASK6 -d fc00::/7 -p tcp --dport 53 -j MARK --set-mark 1
ip6tables -t mangle -A TRANS_PROXY_MASK6 -d fe80::/10 -j RETURN
ip6tables -t mangle -A TRANS_PROXY_MASK6 -d fc00::/7 -j RETURN
ip6tables -t mangle -A TRANS_PROXY_MASK6 -d ::1/128 -j RETURN
ip6tables -t mangle -A TRANS_PROXY_MASK6 -d ff00::/8 -j RETURN
ip6tables -t mangle -A TRANS_PROXY_MASK6 -p tcp -j MARK --set-mark 1
ip6tables -t mangle -A TRANS_PROXY_MASK6 -p udp -j MARK --set-mark 1
ip6tables -t mangle -A OUTPUT -m owner ! --uid-owner tproxy -j TRANS_PROXY_MASK6

ip6tables -t mangle -N TRANS_PROXY6
ip6tables -t mangle -I TRANS_PROXY6 1 -i tailscale0 -j RETURN
ip6tables -t mangle -I TRANS_PROXY6 2 -d fd7a:115c:a1e0::/48 -j RETURN
# 如果劫持 DNS 请求到 sing-box
# ip6tables -t mangle -A TRANS_PROXY6 -d fe80::/10 -p udp --dport 53 -j TPROXY --on-ip 127.0.0.1 --on-port 9898 --tproxy-mark 1
# ip6tables -t mangle -A TRANS_PROXY6 -d fe80::/10 -p tcp --dport 53 -j TPROXY --on-ip 127.0.0.1 --on-port 9898 --tproxy-mark 1
# ip6tables -t mangle -A TRANS_PROXY6 -d fc00::/7 -p udp --dport 53 -j TPROXY --on-ip 127.0.0.1 --on-port 9898 --tproxy-mark 1
# ip6tables -t mangle -A TRANS_PROXY6 -d fc00::/7 -p tcp --dport 53 -j TPROXY --on-ip 127.0.0.1 --on-port 9898 --tproxy-mark 1
ip6tables -t mangle -A TRANS_PROXY6 -d fe80::/10 -j RETURN
ip6tables -t mangle -A TRANS_PROXY6 -d fc00::/7 -j RETURN
ip6tables -t mangle -A TRANS_PROXY6 -d ::1/128 -j RETURN
ip6tables -t mangle -A TRANS_PROXY6 -d ff00::/8 -j RETURN
ip6tables -t mangle -A TRANS_PROXY6 -p udp -j TPROXY --on-ip ::1 --on-port 9898 --tproxy-mark 1
ip6tables -t mangle -A TRANS_PROXY6 -p tcp -j TPROXY --on-ip ::1 --on-port 9898 --tproxy-mark 1
ip6tables -t mangle -A PREROUTING -j TRANS_PROXY6

# 如果劫持 DNS 请求到 mihomo
# ip6tables -t nat -N MIHOMO_DNS6
# ip6tables -t nat -A MIHOMO_DNS6 -p udp --dport 53 -j REDIRECT --to-ports 1053
# ip6tables -t nat -A MIHOMO_DNS6 -p tcp --dport 53 -j REDIRECT --to-ports 1053
# ip6tables -t nat -A OUTPUT -p udp --dport 53 -m owner ! --uid-owner tproxy -j MIHOMO_DNS6
# ip6tables -t nat -A OUTPUT -p tcp --dport 53 -m owner ! --uid-owner tproxy -j MIHOMO_DNS6
# ip6tables -t nat -A PREROUTING -p udp --dport 53 -j MIHOMO_DNS6
# ip6tables -t nat -A PREROUTING -p tcp --dport 53 -j MIHOMO_DNS6
```

清除规则

```shell
# 清理 IPv4 路由
ip rule del fwmark $FWMARK table $TABLE_V4 2>/dev/null || true
ip route del local 0.0.0.0/0 dev lo table $TABLE_V4 2>/dev/null || true

# 清理 IPv4 规则和链 (忽略不存在的错误)
iptables -t mangle -D PREROUTING -p tcp -m socket -j DIVERT 2>/dev/null || true
iptables -t mangle -F DIVERT 2>/dev/null || true
iptables -t mangle -X DIVERT 2>/dev/null || true

iptables -t mangle -D PREROUTING -j ${CHAIN_NAME} 2>/dev/null || true
iptables -t mangle -F ${CHAIN_NAME} 2>/dev/null || true
iptables -t mangle -X ${CHAIN_NAME} 2>/dev/null || true

iptables -t mangle -D OUTPUT -m owner ! --uid-owner $PROXY_USER -j ${CHAIN_NAME}_MASK 2>/dev/null || true
iptables -t mangle -F ${CHAIN_NAME}_MASK 2>/dev/null || true
iptables -t mangle -X ${CHAIN_NAME}_MASK 2>/dev/null || true

# 清理 mihomo DNS 规则
# iptables -t nat -D PREROUTING -p udp --dport 53 -j MIHOMO_DNS 2>/dev/null || true
# iptables -t nat -D PREROUTING -p tcp --dport 53 -j MIHOMO_DNS 2>/dev/null || true
# iptables -t nat -D OUTPUT -p udp --dport 53 -m owner ! --uid-owner $PROXY_USER -j MIHOMO_DNS 2>/dev/null || true
# iptables -t nat -D OUTPUT -p tcp --dport 53 -m owner ! --uid-owner $PROXY_USER -j MIHOMO_DNS 2>/dev/null || true
# iptables -t nat -F MIHOMO_DNS 2>/dev/null || true
# iptables -t nat -X MIHOMO_DNS 2>/dev/null || true

# 清理 IPv6 路由
ip -6 rule del fwmark $FWMARK table $TABLE_V6 2>/dev/null || true
ip -6 route del local ::/0 dev lo table $TABLE_V6 2>/dev/null || true

# 清理 IPv6 规则和链
ip6tables -t mangle -D PREROUTING -p tcp -m socket -j DIVERT6 2>/dev/null || true
ip6tables -t mangle -F DIVERT6 2>/dev/null || true
ip6tables -t mangle -X DIVERT6 2>/dev/null || true

ip6tables -t mangle -D PREROUTING -j ${CHAIN_NAME}6 2>/dev/null || true
ip6tables -t mangle -F ${CHAIN_NAME}6 2>/dev/null || true
ip6tables -t mangle -X ${CHAIN_NAME}6 2>/dev/null || true

ip6tables -t mangle -D OUTPUT -m owner ! --uid-owner $PROXY_USER -j ${CHAIN_NAME}_MASK6 2>/dev/null || true
ip6tables -t mangle -F ${CHAIN_NAME}_MASK6 2>/dev/null || true
ip6tables -t mangle -X ${CHAIN_NAME}_MASK6 2>/dev/null || true

# 清理 mihomo DNS 规则
# ip6tables -t nat -D PREROUTING -p udp --dport 53 -j MIHOMO_DNS6 2>/dev/null || true
# ip6tables -t nat -D PREROUTING -p tcp --dport 53 -j MIHOMO_DNS6 2>/dev/null || true
# ip6tables -t nat -D OUTPUT -p udp --dport 53 -m owner ! --uid-owner $PROXY_USER -j MIHOMO_DNS6 2>/dev/null || true
# ip6tables -t nat -D OUTPUT -p tcp --dport 53 -m owner ! --uid-owner $PROXY_USER -j MIHOMO_DNS6 2>/dev/null || true
# ip6tables -t nat -F MIHOMO_DNS6 2>/dev/null || true
# ip6tables -t nat -X MIHOMO_DNS6 2>/dev/null || true
```

更加便捷的应用和清除规则可以参考[脚本](https://github.com/zhiim/dotfiles/blob/master/dot_config/sing-box/tproxy.sh)。
