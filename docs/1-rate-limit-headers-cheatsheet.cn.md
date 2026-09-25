---
x-title: 速率限制响应头 —— X-RateLimit-* / Retry-After / cf-mitigated / RateLimit-* 草案
x-desc: 跨服务速率限制响应头对照：GitHub 发 X-RateLimit-* 全套（Reset 是 UNIX epoch）；Cloudflare 发 Retry-After + cf-mitigated（不发 X-RateLimit-*）；Bing/Yahoo/DuckDuckGo/Baidu/Shenma 多为裸 429；IETF RateLimit-* 草案是无 X- 前缀的新规范。
x-sidebar: 速率限制响应头速查
x-keywords: 速率限制, ratelimit, response headers, retry-after, x-ratelimit, cf-mitigated, http headers, 429, 403, 跨服务, 跨厂商
x-json-ld:
  '@context': https://schema.org
  '@graph':
    - '@type': TechArticle
      headline: '速率限制响应头速查'
      inLanguage: 'zh-CN'
      about: '跨服务限速响应头对比'
---

# 速率限制响应头 —— 跨服务速查

读速率限制响应，看 **HTTP response headers**。不同服务用不同 header 家族，但模式都差不多。

---

## 速查

### 各服务用什么 header 告诉你限速

| 服务 | 状态码 | Header | 含义 |
| --- | --- | --- | --- |
| **GitHub** | 200 / 403 | `X-RateLimit-Limit` / `Remaining` / `Reset` / `Used` / `Resource` | 当前桶的限速状态（每次响应都有） |
| **GitHub** | 429 | `Retry-After` | 整数秒，等这么久再试 |
| **Cloudflare** | 429 | `Retry-After` | 整数秒 |
| **Cloudflare** | 200 / 403 | `cf-mitigated: challenge\|block\|rate-limit\|bot\|ip\|country` | WAF / Bot 检测信号 |
| **Cloudflare** | 任意 | `cf-ray` | 内部追踪 ID，找客服时给这个 |
| **Bing / IndexNow** | 429 | 通常裸 429，可能带 `Retry-After` | — |
| **Yahoo** | 429 | 通常裸 429 | — |
| **DuckDuckGo** | 429 | 通常裸 429 | — |
| **Baidu** | 429 | 通常裸 429 | — |
| **Shenma** | 429 | 通常裸 429 | — |

**关键模式**：

- **`Retry-After` 是跨服务公约**——多数服务遵守 HTTP 标准
- **`X-RateLimit-*` 是 GitHub 家族**——其他服务基本不用
- **`cf-mitigated` 是 Cloudflare 特有**——只在 Cloudflare edge 出现
- **Bing / Yahoo / DuckDuckGo / Baidu / Shenma**：大多只返 429 状态码，详细 header 少

### Header 家族对照

| 家族 | 谁用 | 字段 |
| --- | --- | --- |
| `X-RateLimit-*` | GitHub | `Limit` / `Remaining` / `Reset` / `Used` / `Resource` |
| `Retry-After`（HTTP 标准） | GitHub / Cloudflare / 多数 | 整数秒 |
| `cf-mitigated` | Cloudflare | `challenge` / `block` / `rate-limit` / `bot` / `ip` / `country` |
| `cf-ray` | Cloudflare | 追踪 ID |
| `RateLimit-*`（RFC 标准草案） | 少量新服务 | `limit` / `remaining` / `reset` / `policy` |

**趋势**：行业在向 IETF `RateLimit-*` 草案头靠拢，但 GitHub 的 `X-RateLimit-*` 和 Cloudflare 的 `cf-mitigated` 都还是各家自己的实现。

---

## 正文

### GitHub：`X-RateLimit-*` 家族

每次响应都带（不仅是 429）：

```
HTTP/1.1 200 OK
X-RateLimit-Limit: 5000
X-RateLimit-Remaining: 4999
X-RateLimit-Reset: 1640000000
X-RateLimit-Used: 1
X-RateLimit-Resource: core
```

- **`X-RateLimit-Reset` 是 UNIX 时间戳**（不是"还剩 N 秒"）。算剩余时间：`wait = max(reset - now, 1)`
- **`X-RateLimit-Resource`** 告诉你当前扣的是哪个桶（`core` / `search` / `graphql` / `integration_manifest` 等）
- **`Retry-After`** 只在 429 上有——是相对秒数

完整说明见 [2-github](2-github) §三。

### Cloudflare：`Retry-After` + `cf-mitigated`

Cloudflare 的速率限制和 WAF / Bot 用两套 header：

```
HTTP/1.1 429 Too Many Requests
Retry-After: 60
```

```
HTTP/1.1 403 Forbidden
cf-mitigated: challenge
cf-ray: 6c8a...
```

- **`429` + `Retry-After`**：真·速率限制（配额到了）
- **`cf-mitigated: challenge|block|bot|ip|country`**：WAF / Bot 触发——**不是速率限制**
- **`cf-ray`**：联系 Cloudflare 客服时给这个 ID，能查到具体触发原因

**关键**：Cloudflare **不发** `X-RateLimit-*` 家族——不要去找它。

完整说明见 [3-cloudflare](3-cloudflare) §一。

### Bing / IndexNow / Yahoo / DuckDuckGo / Baidu / Shenma

搜索引擎（除 Google）大多简单粗暴：

- **429** 状态码（偶尔 503）
- 可能带 `Retry-After`，可能不带
- **不**带详细 `X-RateLimit-*` family

实操建议：
- 撞 429 就 sleep 一段时间（30 秒到几分钟）
- 保守并发（≤ 1 QPS per IP）
- 用代理 / IP 池绕开单 IP 限制

详细数字见各服务专门文章：[3-cloudflare](3-cloudflare) / 后续 Bing / Yahoo / DuckDuckGo / Baidu / Shenma 等。

### IETF `RateLimit-*` 草案

行业正在向 IETF 的 `RateLimit-*` 家族靠拢：

```
RateLimit-Limit: 100
RateLimit-Remaining: 50
RateLimit-Reset: 30
```

- 字段和 GitHub 类似但**无 `X-` 前缀**
- `Reset` 是**相对秒数**（不像 GitHub 的绝对时间戳）
- 目前只有少量新服务用，主流服务还在用自家实现

---

## 怎么读一个限速响应（实战 5 步）

不管哪个服务，读到 429 / 403 时：

1. **看状态码** —— 429 vs 403 vs 200-with-cf-mitigated
2. **看 `Retry-After`** —— 有就是直接告诉等多久
3. **看 `X-RateLimit-*`**（如果有）—— 算剩余时间和 reset 时间
4. **看 `cf-mitigated`**（如果服务是 Cloudflare 后面的）—— 判断是速率限制还是 WAF
5. **看 `cf-ray`** 或服务特定追踪 ID —— 找客服时给这个能加快排查

---


---

## 来源

- GitHub rate-limit headers: <https://docs.github.com/en/rest/using-the-rest-api/rate-limits-for-the-rest-api>
- Cloudflare protections / headers: <https://developers.cloudflare.com/fundamentals/reference/protections/>
- IETF RateLimit headers draft: <https://datatracker.ietf.org/doc/draft-ietf-httpapis-ratelimit-headers/>

---

## 你想要什么 ratelimit 速查？（征集反馈）

这篇文章**故意排在第二篇**——它是你撞了 429 / 403、不管是哪家服务都能直接查的速查表。团队正在积极扩展。

如果有**没覆盖到的速查需求**，**先开 issue**——issue 优先、PR 殿后是我们推荐的流程：

| 你想做什么 | 开这种 issue |
| --- | --- |
| 把新的服务 / API 表面加进表里 | [`monitor-target`](https://github.com/x-cmd/ratelimit/issues/new?template=monitor-target.yml) |
| 修一个错的 header 名 / 状态码 / 描述 | [`errata`](https://github.com/x-cmd/ratelimit/issues/new?template=errata.yml) |
| 建议新维度（WebSocket 429？GraphQL 二次限速？）或其它任何事 | [`other`](https://github.com/x-cmd/ratelimit/issues/new?template=other-suggestion.yml) |

如果这里**哪段写错了**、你在生产里撞到才发现——429 风暴、明明不是速率限制却来了 `cf-mitigated`、header 字段行为跟文档对不上——**开 `errata`**。最快的修法就是一个 issue，里面写清：哪一家、症状、官方文档链接。

> **AI agent 创建 issue**：每个模板都以 `type: <monitor-target | errata | other 三选一>` 开头，下面有一段提示。**先读提示**；每条声明都附官方文档 URL；**maintainer 确认范围之前不要开 PR**。