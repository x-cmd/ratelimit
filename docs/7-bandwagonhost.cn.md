---
x-title: BandwagonHost 速率限制 —— VPS 端口、带宽、连接上限
x-desc: "BandwagonHost 的 VPS 级速率等价物：默认屏蔽出站 25 端口（SMTP）、月带宽上限、网卡速率、软连接并发上限；以及缺乏公开 API 速率表这件事。"
x-sidebar: BandwagonHost 速率限制
x-keywords: bandwagonhost, vps, 端口 25, smtp 带宽, 连接上限, it7 networks
x-json-ld:
  '@context': https://schema.org
  '@graph':
    - '@type': TechArticle
      headline: 'BandwagonHost 速率限制'
      inLanguage: 'zh-CN'
      about: 'BandwagonHost 的 VPS 级速率等价物'
---

# BandwagonHost 速率限制 —— VPS 端口、带宽、连接上限

BandwagonHost（现属 IT7 Networks）是 **VPS 主机商**，不是
API 服务。对 Cloudflare / GitHub / Vercel 那套"速率限制"框架
并不完全适用；真正成立的是**连接级与出站流量约束**，你的工作
负载若推到极限，那种感觉确实像速率限制。

## 出站端口 25（SMTP）—— 头条约束

| 面 | 上限 | 备注 |
| --- | --- | --- |
| 出站 TCP/25（SMTP） | **默认屏蔽** | 所有 VPS 套餐，按 BandwagonHost TOS |

SMTP 出站是 BandwagonHost VPS 上最常见的"邮件发不出去"原因。
屏蔽是为了防止 IP 段被滥用为垃圾邮件源。

**绕过方案：**

1. **通过 support 申请解封。** 说明用途（已知应用的事务邮
   件，不是群发）。BandwagonHost 按客户审后可解封。这是最干
   净的路。
2. **用事务邮件提供商。** SendGrid / Mailgun / Postmark /
   Amazon SES 都提供 SMTP 端点，你的 VPS 连 587（或 465 +
   TLS）端口 —— **不**被屏蔽。提供商负责实际投递。
3. **用 API-based 提供商。** Mailgun / Postmark / SendGrid /
   SES 都有 HTTP API，**完全不用 SMTP**。你的 VPS 调 HTTPS，
   提供商发邮件。

如果你真要直出 SMTP —— **申请解封**。试图通过自定义端口或
SMTP-relay-via-Gmail 绕过都是脆弱且违反 TOS 的。

## 带宽上限

| 套餐档次 | 月出站流量 | 网卡速率 |
| --- | --- | --- |
| 入门 VPS（1 GB RAM、20 GB SSD） | 1 TB / 月 | 1 Gbps |
| 中端 VPS（2 GB RAM、40 GB SSD） | 2 TB / 月 | 1 Gbps |
| 特殊 / 更大套餐 | 视套餐 | 10 Gbps（部分） |

这些是**软上限** —— BandwagonHost 在你接近上限时会通知，大
套餐可能超额收费。硬性节流不常见。

如果你的 VPS 公开服务（公共服务、大文件托管），监控月流量并
围绕上限做规划。

## 并发连接

| 面 | 上限 | 备注 |
| --- | --- | --- |
| 每 VPS 并发 TCP 连接 | 视情况，软上限 | 与套餐相关；无公开硬数 |

BandwagonHost 不公开每套餐的连接上限。实际瓶颈通常是：

- VPS 的 RAM（每个连接消耗一些内核状态）
- 共享端口速率（1 Gbps ≈ HTTP/1.1 keepalive 下 10k 并发
  请求/秒）

如果你的工作负载撞连接瓶颈，**升级 VPS 套餐**比等具体上限
被放宽更有效。

## 控制面板 API

BandwagonHost 客户端区（管理 VPS 实例的地方）有自己的内部
API 速率限制，但**不公开**且**不鼓励自动化**。他们的 TOS 明
确指出客户端区的自动化访问会触发人工审核。

编程管理 BandwagonHost VPS 时，**用 SolusVM API**（他们的
控制面板后端），如果你的套餐提供；或者**用 SSH + 配置管理工具**
（Ansible / SaltStack / bash）在 VPS 本地做。

## BandwagonHost 速率限制故事不涵盖的

本文不涵盖：

- DDoS 防护（BandwagonHost TOS 按滥用报告逐案处理）
- 入站流量（无上限；这是主机商的常规做法）
- CPU / RAM 限制（这些是 VPS 套餐规格，不是速率限制）
- 存储 I/O 限制（视套餐）

这些是不同话题；见 BandwagonHost TOS 与套餐对比页。

## 来源

- 服务条款（端口 25 屏蔽、带宽上限、TOS 强制执行）：
  <https://bandwagonhost.com/terms.php>
- 套餐与价格（每档带宽上限、网卡速率）：
  <https://bandwagonhost.com/cart.php>
- 网络与数据中心信息：
  <https://bandwagonhost.com/>
- SolusVM API（控制面板后端）—— 第三方文档：
  <https://docs.solusvm.com/>
- IT7 Networks（母公司）：
  <https://www.it7.net/>

**核实状态**：待 CI scraper。端口 25 屏蔽与带宽上限源自
TOS；每套餐具体数字需与实时购物车页面比对。
