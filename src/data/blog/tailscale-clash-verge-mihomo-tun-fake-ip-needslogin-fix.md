---
author: Cola
pubDatetime: 2026-03-02T10:30:00+08:00
title: Tailscale + Clash Verge（Mihomo）排障记录：修复 TUN + fake-ip 导致的 NeedsLogin
slug: tailscale-clash-verge-mihomo-tun-fake-ip-needslogin-fix
featured: false
draft: false
tags:
  - Tailscale
  - ClashVerge
  - Mihomo
  - Networking
  - Troubleshooting
description: 一次 Tailscale 与 Clash Verge（Mihomo）共存时的排障记录，定位并修复 TUN + fake-ip 导致 controlplane.tailscale.com 被污染，最终引发 tailscale up 卡住与 NeedsLogin 问题。
---

在 macOS 上同时使用 Tailscale 和 Clash Verge（Mihomo）时，如果 Clash 开启了 `TUN + fake-ip`，可能会遇到 Tailscale 无法正常登录控制面的问题。下面是一次完整可复现、可回滚的修复记录。

## 现象

- `tailscale up` 长时间等待或超时。
- `tailscale status` 显示 `Logged out` / `NeedsLogin`。

## 根因

- Clash Verge 开启 `TUN + fake-ip` 后，`controlplane.tailscale.com` 被解析成 fake-ip（常见为 `198.18.x.x`）。
- Tailscale 控制面注册/登录链路异常，最终表现为登录状态失效或持续卡住。

## 标准修复（保留 TUN 模式）

使用 Clash Verge 的全局扩展脚本（Global Extension Script），让当前和后续导入的订阅都自动生效。

```javascript
function main(config, profileName) {
  config.dns = config.dns || {};
  const key = "fake-ip-filter";
  const fakeIpFilter = Array.isArray(config.dns[key]) ? config.dns[key] : [];

  const domains = ["+.tailscale.com", "+.ts.net"];
  for (const d of domains) {
    if (!fakeIpFilter.includes(d)) fakeIpFilter.push(d);
  }
  config.dns[key] = fakeIpFilter;

  if (!Array.isArray(config.rules)) config.rules = [];

  const tailRules = [
    "PROCESS-NAME,tailscaled,DIRECT",
    "PROCESS-NAME,tailscale,DIRECT",
    "DOMAIN-SUFFIX,tailscale.com,DIRECT",
    "DOMAIN-SUFFIX,ts.net,DIRECT",
    "IP-CIDR,100.64.0.0/10,DIRECT,no-resolve"
  ];

  config.rules = config.rules.filter((r) => !tailRules.includes(r));
  config.rules.unshift(...tailRules);

  return config;
}
```

## Clash Verge 配置位置

1. 打开 Clash Verge。
2. 进入全局扩展脚本（Global Extension Script）。
3. 粘贴上述脚本。
4. 点击应用配置/重载内核。

## 修改后执行

```bash
sudo dscacheutil -flushcache
sudo killall -HUP mDNSResponder
tailscale up
```

## 验证

先检查控制面域名解析结果：

```bash
dscacheutil -q host -a name controlplane.tailscale.com
```

- 期望结果：不再是 `198.18.x.x`。

再检查 Tailscale 状态：

```bash
tailscale status
```

- 期望结果：已登录，不再出现 `NeedsLogin`。

## 问题复发时的快速诊断命令

```bash
tailscale status --json | sed -n '1,120p'
tailscale netcheck | sed -n '1,120p'
dscacheutil -q host -a name controlplane.tailscale.com
```

## 回滚

删除全局扩展脚本中的上述规则，重新应用配置并重载内核即可。
