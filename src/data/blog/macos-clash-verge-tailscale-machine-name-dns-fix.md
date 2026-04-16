---
author: Cola
pubDatetime: 2026-04-16T12:00:00+08:00
title: macOS + Clash Verge + Tailscale：修复通过机器名访问服务不稳定的问题
slug: macos-clash-verge-tailscale-machine-name-dns-fix
featured: false
draft: false
tags:
  - macOS
  - Tailscale
  - ClashVerge
  - DNS
  - Networking
  - Troubleshooting
description: 在 macOS 上同时使用 Clash Verge 和 Tailscale 时，通过机器名访问服务偶发 Could not resolve host，甚至被错误解析到 198.18.x.x。本文记录如何通过 /etc/resolver/ts.net 把 *.ts.net 查询定向交给 Tailscale DNS，从根上修复问题。
---

最近排查了一个挺迷惑的问题：`tailscale ping` 明明是通的，直接访问 Tailscale IP 也没问题，但一旦改成通过机器名访问服务，就时灵时不灵。

更怪的是，失败时并不总是一个症状。有时是单纯的 `Could not resolve host`，有时则会被解析到奇怪的 `198.18.x.x` 地址，看起来像是 Tailscale 出了问题，实际根因却在本机 DNS 链路。

文中示例已经做过脱敏处理，机器名、Tailnet 后缀、IP 和端口都不是实际值。

## 现象

我的现场表现大概是这样：

- `tailscale ping <device-name>` 可以正常连通。
- 直接访问 Tailscale IP 可以通。
- 通过机器名或完整 `*.ts.net` 名称访问服务不稳定。

例如：

```bash
curl -I http://100.x.y.z:6000
curl http://my-node:6000
curl http://my-node.tailxxxx.ts.net:6000
```

失败时常见报错是：

```text
Could not resolve host
```

或者域名被错误解析到 `198.18.x.x` 这样的地址。

## 先说结论

这类问题很多时候不在 Tailscale 网络本身，而在 macOS 这一侧的 DNS 分发出了岔子。

在我这次的环境里，同时满足下面几个条件：

- 开着 Clash Verge。
- Clash 的全局扩展配置或脚本改写了 DNS 行为。
- macOS 没有把 `*.ts.net` 的查询稳定交给 Tailscale 本地 DNS `100.100.100.100`。

于是结果就变成了：

- Tailscale 网络本身是通的。
- 但系统层面对 `my-node` 或 `my-node.tailxxxx.ts.net` 的解析失败。

## 根因

Tailscale 的设备访问，尤其是基于 MagicDNS 和 `*.ts.net` 的命名访问，本质上依赖 DNS 正常工作。

如果本机 DNS 查询先被 Clash Verge 接管，又被某些扩展规则、脚本或 fake-ip 逻辑干扰，那么 `*.ts.net` 查询就可能：

- 直接解析失败；
- 没有交给 `100.100.100.100`；
- 或者落到不该出现的 `198.18.x.x` 假地址上。

这时候你会看到一个很误导人的现象：`tailscale ping` 没问题，但浏览器、`curl`、应用内 URL 却打不开。

## 正确修复方法

这里不建议走两种常见但不够稳的路：

- 不要把真实设备名硬写进 `/etc/hosts`。
- 也不要把整机 Wi-Fi DNS 粗暴改成 `100.100.100.100`。

更稳的做法是只对 `ts.net` 这个后缀做定向解析，让 macOS 知道：

> 只有 `*.ts.net` 这一类请求，才交给 Tailscale 的本地 DNS。

执行下面几条命令：

```bash
sudo mkdir -p /etc/resolver
printf "nameserver 100.100.100.100\n" | sudo tee /etc/resolver/ts.net >/dev/null
sudo dscacheutil -flushcache
sudo killall -HUP mDNSResponder
```

做完之后，macOS 会把 `*.ts.net` 的解析交给 Tailscale，本机其他域名查询则继续走你原本的 DNS 配置。

## 验证方法

建议先验证完整域名，再验证短名字。

先测完整域名：

```bash
ping -c 1 my-node.tailxxxx.ts.net
curl http://my-node.tailxxxx.ts.net:6000
```

如果这两个已经恢复，再试短名字：

```bash
ping -c 1 my-node
curl http://my-node:6000
```

如果你开着 MagicDNS，短名字通常也会一起恢复；如果没有，优先使用完整的 `*.ts.net` 域名会更稳。

## 为什么这一招有效

原因其实很直接：

- `100.100.100.100` 是 Tailscale 提供的本地 DNS。
- `/etc/resolver/ts.net` 告诉 macOS：凡是 `*.ts.net` 的查询，都走这台 DNS。
- 这样只修正了 Tailscale 相关域名，不会污染全局 DNS 行为。

这也是它比直接改 `/etc/hosts` 更适合长期使用的原因。

## 为什么不推荐 `/etc/hosts`

`/etc/hosts` 看起来简单，但它的问题也很明显：

- 它会把机器名硬绑定到某一个 IP。
- Tailscale 设备 IP 变化后，这条记录就会失效。
- 这种静态绑定不适合动态网络。

相比之下，`ts.net -> 100.100.100.100` 是动态解析方案，设备 IP 变化时不需要你手工维护。

## 与 Clash Verge 共存的建议

我最后保留下来的状态是：

- Clash Verge 继续正常开着。
- 全局扩展复写配置清空。
- 全局扩展脚本清空，或者只保留空实现。
- macOS 增加 `/etc/resolver/ts.net`。

这样做的好处是边界更清楚：

- Clash 负责它该负责的代理流量。
- Tailscale 负责它自己的设备命名和内网解析。

两边各干各的，反而最稳。

## 日常使用建议

如果只是自己收藏书签，短地址更顺手：

```text
http://my-node:6000
```

如果想要更标准、也更利于排障，建议直接保存完整域名：

```text
http://my-node.tailxxxx.ts.net:6000
```

## 以后再失效，先查这两项

先看 resolver 文件是否还在：

```bash
cat /etc/resolver/ts.net
```

正常应该看到：

```text
nameserver 100.100.100.100
```

再看 Tailscale DNS 是否还能正确解析：

```bash
dig +short @100.100.100.100 my-node.tailxxxx.ts.net
```

如果这里能返回一个 `100.x.y.z` 地址，说明 Tailscale 本身大概率没问题，重点就该回到本机 DNS 分发链路继续查。

## 小结

这次问题最容易让人绕进去的地方，是“网络通”和“名字能解析”并不是一回事。

Tailscale 连通，不代表 macOS 一定把 `*.ts.net` 的查询正确交给了 Tailscale DNS。只要本机 DNS 链路被 Clash Verge 的配置、脚本或 fake-ip 逻辑干扰，就可能出现“IP 能访问、机器名却不行”的割裂现象。

如果你也碰到类似问题，优先试试给 macOS 增加 `/etc/resolver/ts.net`。这个修法足够克制，也更接近这类问题真正的根。
