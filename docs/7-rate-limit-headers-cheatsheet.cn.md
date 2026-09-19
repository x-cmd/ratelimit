---
x-title: 速率响应头速查 —— 跨厂商
x-desc: 跨厂商的 HTTP 速率响应头约定 —— `Retry-After`、`X-RateLimit-*`、IETF `RateLimit-*` draft（RFC 9745），以及状态码（429 与自定义 400/403）、窗口语义、重置语义的差异。
x-sidebar: 速率响应头速查
x-keywords: 速率限制响应头, ratelimit-limit, ratelimit-remaining, retry-after, x-ratelimit, rfc 9745, 429
x-json-ld:
  '@context': https://schema.org
  '@graph':
    - '@type': TechArticle
      headline: '速率响应头速查'
      inLanguage: 'zh-CN'
      about: '跨厂商 HTTP 速率响应头约定'
---

# 速率响应头速查 —— 跨厂商

不同厂商实现速率响应头的形状各异。本文是交叉参考 —— 什么是
通用的，什么是各家自定义，IETF draft（RFC 9745）说它应该长什么样。

## 响应头速查表

| 响应头 | 含义 | IETF 标准？ |
| --- | --- | --- |
| `Retry-After` | 下次请求的等待秒数 | RFC 6585（通用） |
| `X-RateLimit-Limit` | 当前窗口配额上限 | IETF draft 用 `RateLimit-Limit`（无 `X-` 前缀） |
| `X-RateLimit-Remaining` | 窗口剩余请求数 | IETF draft 用 `RateLimit-Remaining` |
| `X-RateLimit-Reset` | **Epoch 时间戳**（非秒数） | IETF draft 用 `RateLimit-Reset` |
| `X-RateLimit-Resource` | 哪个 bucket（`core`、`search` 等） | GitHub 专属 |
| `ratelimit-limit` | Vercel 小写 | IETF draft 合规 |
| `ratelimit-remaining` | Vercel 小写 | IETF draft 合规 |
| `ratelimit-reset` | Vercel 小写 | IETF draft 合规 |
| `RateLimit` | 组合头：`limit=100, remaining=50, reset=60` | IETF draft |
| `cf-mitigated` | Cloudflare 专属滥用信号 | 不是速率限制 |

## 按厂商分

| 厂商 | 标准 `Retry-After`？ | 自定义响应头？ | 信任语义 |
| --- | --- | --- | --- |
| Cloudflare | 是（多数端点） | `cf-mitigated` | API 配额在各套餐间统一 |
| 阿里云 | **否**（返回 400/403） | body 里 `Code` 字段 | 三档限流：User / Api / CloudBox |
| 腾讯云 | 是（整数秒） | `X-RateLimit-*` | 这里所有厂商里最贴近 IETF draft |
| GitHub | 是（429 时） | `X-RateLimit-*`、`X-RateLimit-Resource` | 双 bucket：主限 + 二级 |
| Vercel | 是（小写） | `ratelimit-*` | RFC 9745 风格响应头 |
| BandwagonHost | n/a | n/a | 无程序化 API |

## IETF draft（RFC 9745）

IETF draft 在 2024 年成为 RFC 9745，定义了一个合并头：

```
HTTP/1.1 429 Too Many Requests
RateLimit: limit=100, remaining=50, reset=60
Retry-After: 60
```

- `RateLimit:`（单一响应头）—— 三个逗号分隔的 key=value：
  `limit`、`remaining`、`reset`
- `Retry-After:` —— 秒数（整数）或 HTTP-date
- `reset` —— 距离重置的秒数（**不是** epoch 时间戳 —— 这是
  对 `X-RateLimit-Reset` 的关键修正）

Vercel 最贴近这种格式（小写 `ratelimit-*` 响应头按端点）。
其他多数厂商还在用历史上的 `X-RateLimit-*` 风格 + epoch 时间戳。

## 客户端代码三条经验

1. **有 `Retry-After` 就解析它。** 这是最便携的信号：每个会
   发响应头的厂商都发这个。（阿里云例外 —— 他们根本不发任何
   标准速率响应头。）
2. **优先信任厂商特定信号，再信任通用信号。** 当 GitHub 返回
   429 + `Retry-After: 60` 且 `X-RateLimit-Resource: search`，
   撞的是 search 配额，不是 core REST 配额。
3. **本地跟踪预算，不要靠重试探测。** `X-RateLimit-Remaining`
   是权威但你不必为查它发请求 —— 本地记数器减 1 即可。省掉
   "启动时我的配额是多少？"的引导难题。

## 为什么要关心这件事

健壮的 HTTP 客户端需要处理六家厂商的响应头形态 —— 阿里云
"无响应头、自定义 `Code` 字段"是最容易忘的，直到被咬一口。
上表的速查是任何宣称"多厂商速率限制支持"的客户端库必须测试
通过的最小覆盖面。