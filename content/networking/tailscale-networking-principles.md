---
title: Tailscale 组网原理 — 学习笔记
created: 2026-08-05
updated: 2026-08-05
tags: ["tailscale", "networking", "vpn", "wireguard", "网络"]
category: networking
source: /Users/xuebao/Documents/Tailscale网络原理学习笔记.html
---

# Tailscale 组网原理 — 学习笔记

基于「量化交易系统部署到 Vultr VPS」实战 · 从配置操作到网络原理逐层拆解

### 📑 目录

- 01 · 操作清单回顾
- 02 · 核心概念：什么是 Tailscale
- 03 · 分层原理
- 04 · 一条请求的完整旅程
- 05 · 架构要点总结
- 06 · 学习资源

## 01 · 操作清单回顾

以下是本次部署 VPS 时执行的全部 Tailscale 相关操作：

| 操作 | 作用 | 涉及原理 |
|---|---|---|
| `curl .../install.sh \| sh` | 安装 Tailscale 客户端 + `tailscaled` 守护进程 | 用户态进程 + 内核 WireGuard 模块 |
| `tailscale up --hostname multiagent-vps` | 登录认证，把 VPS 加入你的私有网络（tailnet） | OAuth 式授权、设备身份注册、密钥交换 |
| `tailscale status` | 查看网络内所有设备及其 Tailscale IP | 控制平面设备清单同步 |
| `tailscale cert ...ts.net` | 为 HTTPS 签发 TLS 证书 | ACME / Let's Encrypt 自动证书 |
| `tailscale serve --bg http://127.0.0.1:8501` | 443 端口反向代理，转发到本机 dashboard | 反向代理 + tailnet-only 访问控制 |
| ufw 规则（443/8501 仅限 100.64.0.0/10、41641/udp） | 防火墙配合：只放行隧道内流量 | CGNAT 地址段识别 + 纵深防御 |

## 02 · 核心概念：什么是 Tailscale

**一句话**：Tailscale 是一个基于 WireGuard 的现代组网工具，它在你所有设备之间织一张"私有的叠加网络"（overlay network）——不管设备在哪（家里、手机流量、云服务器），都像在同一个局域网里。

### 三个关键概念

- **tailnet**：你的私有网络（"尾巴网"），只有你授权的设备能加入。每台设备像"居民"，协调服务器像"门卫"。

- **Tailscale IP**：`100.x.x.x`，属于 **100.64.0.0/10** 保留段（RFC 6598 的 CGNAT 地址段，运营商级 NAT 专用）。公网上不可能出现这个地址段的真实主机——这就是为什么防火墙规则 `ALLOW 100.64.0.0/10` 等于"只允许隧道流量"。

- **tailscaled 守护进程**：每台设备上常驻的进程，创建虚拟网卡 `tailscale0`，负责加密、路由、DNS 解析。

## 03 · 分层原理

### 3.1 WireGuard — 加密隧道（数据平面）

WireGuard 是 Linux 内核自带的 VPN 协议，比 OpenVPN/IPSec 快 10 倍以上、代码仅数千行。核心特性：

- **密钥对**：每台设备一对（公钥/私钥），握手后建立加密通道，相互认证

- **UDP 41641 端口**：隧道流量走 UDP —— 所以防火墙开 `41641/udp`

- **加密**：流量进入虚拟网卡即被加密，穿过任何网络（路由器/运营商/云厂商）都看不到内容

### 3.2 控制平面 vs 数据平面 — 关键架构

┌─────────────────────────────────────────────┐
│  Tailscale 协调服务器 (coordination server)  │ ← 控制平面
│  只做: 身份认证 / 设备发现 / 公钥交换 / ACL  │
└─────────────────────────────────────────────┘
        ↑ 持久连接(仅元数据)        ↑ 设备上线/下线通知
   ┌────────┐                ┌────────┐
   │  手机   │◄──── WireGuard 直连/中继 ────►│  VPS   │  ← 数据平面
   └────────┘                └────────┘

- **控制平面**：协调服务器只管"谁是谁、谁的钥匙是什么、谁能连谁"。设备登录后与其保持一条轻量连接，实时同步设备清单。它**不转发你的业务流量**。

- **数据平面**：真实流量（如打开 dashboard 的请求）**不经过协调服务器**，设备间点对点直连。协调服务器即使被攻破也看不到你的数据。

### 3.3 NAT 穿越 — 任意网络下为何能直连

手机在蜂窝网络（运营商 NAT 后），VPS 在云机房，中间隔着层层 NAT。Tailscale 用"三板斧"建立连接：

| 技术 | 原理 | 类比 |
|---|---|---|
| STUN 探测 | 通过协调服务器交换各自的公网出口 IP:端口（NAT 映射信息） | 互相告诉对方"我的门牌号是啥" |
| UDP 打洞 | 双方同时向对方推测的映射地址发包；NAT 会放行"自己刚发过包的对方地址"来的包 | 同时向对方大喊，两边门卫都放行"刚听过的声音" |
| DERP 中继（兜底） | 打洞失败（对称 NAT / 运营商限制）时走全球中继节点转发，中继只转发密文、看不到内容 | 直连不了就走"公共邮局"（加密信封） |

💡 这就是为什么开 `41641/udp`：让打洞直连尽量成功，避免流量退化到中继（中继稍慢且依赖 Tailscale 基础设施）。

### 3.4 MagicDNS — 不用记 IP 的域名解析

- tailscaled 内置 DNS 解析器（`100.100.100.100`），自动把主机名解析成 Tailscale IP

- 每台设备自动获得域名 `主机名.<tailnet名>.ts.net` —— 你访问的 `multiagent-vps.tail65040b.ts.net` 就是这么来的，`tail65040b` 是你 tailnet 的唯一标识

### 3.5 Tailscale Serve — 本机服务的"安全前门"

手机 ──► https://multiagent-vps.tail65040b.ts.net:443 (加密)
                │
                ▼
          tailscaled (serve 反向代理, 只服务 tailnet 设备)
                │
                ▼ 转发到本机
          http://127.0.0.1:8501 ──► Streamlit dashboard

- **反向代理**：把 443 端口的 HTTPS 请求转发给本机任意端口（本例 8501）

- **自动 HTTPS**：`tailscale cert` 从 Let's Encrypt 拿 ts.net 域名的可信证书（admin console 的 *"HTTPS Certificates"* 开关 = 允许 Tailscale 替你管理证书签发）

- **tailnet only**：只有你网络内的设备能访问，外部连接被拒绝 —— "只给认识的人开门"

- 好处：本地服务（dashboard）代码零改动，无需把端口暴露公网

### 3.6 防火墙协同 — 纵深防御

公网攻击者 ──► VPS:443/8501 → ❌ 被 ufw 拒绝 (源 IP 不是 100.64.0.0/10)
你的手机  ──► 100.80.9.89:443 → ✅ 源 IP=100.x (WireGuard 解密后) 放行

原理：流量穿过 WireGuard 隧道到达 VPS 时，从虚拟网卡出来，源地址已是 `100.x.x.x`；而公网直连的源地址是普通公网 IP。因此 `ALLOW 100.64.0.0/10` 数学上**只可能放行隧道内流量**——即使误开了端口，公网也进不来，这是两层保险。

## 04 · 一条请求的完整旅程

你在手机浏览器输入 `https://multiagent-vps.tail65040b.ts.net` 后发生的一切：

  1. MagicDNS 解析 → 100.80.9.89 (手机 Tailscale App 内置解析器)
  2. 流量进入手机虚拟网卡 tailscale0 → WireGuard 加密
  3. 加密包打洞直连(或 DERP 中继) → 到达 VPS 的 UDP 41641
  4. VPS tailscaled 解密 → 虚拟网卡输出"原始 HTTPS 请求"
  5. tailscale serve 监听 443 收到请求 → 校验设备身份(属于你的 tailnet)
  6. 转发到 http://127.0.0.1:8501 → Streamlit 响应
  7. 响应原路加密返回手机浏览器

## 05 · 架构要点总结

| 设计思想 | 落地方式 |
|---|---|
| **默认安全 (Zero Trust)** | 公网零端口暴露；只有 tailnet 白名单设备可访问；服务本身仍带密码登录（纵深防御） |
| **控制/数据分离** | 协调服务器只管元数据，业务流量点对点直连加密 |
| **尽力直连、中继兜底** | STUN + UDP 打洞 + DERP，保证任何 NAT 环境下连通 |
| **标准协议复用** | WireGuard（加密）+ Let's Encrypt（证书）+ RFC 6598（地址段） |
| **应用无感** | dashboard 仍监听 127.0.0.1，serve 反向代理透明转发 |

## 06 · 学习资源

- **Tailscale 官方文档**：tailscale.com/kb — 重点看 *"How Tailscale works"* 和 *"NAT traversal"*

- **WireGuard 白皮书**：wireguard.com/whitepaper — 加密协议设计（论文级但可读）

- **NAT 打洞原理**：搜 "UDP hole punching"，配合 RFC 5128（NAT traversal 综述）

**动手实验建议**：

```
tailscale ping multiagent-vps   # 看直连(direct)还是中继(relay)
tailscale netcheck              # 看本机 NAT 类型和 UDP 连通性
tailscale serve status          # 看 HTTPS 代理配置
tailscale status --json         # 看完整设备信息(含 DNSName)
```

  本文档基于 2026-08 实战部署整理 · 量化交易系统 VPS 部署项目

  关键词：WireGuard / overlay network / NAT traversal / MagicDNS / Tailscale Serve

