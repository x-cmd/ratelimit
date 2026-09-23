---
x-title: Cloudflare 报错时怎么区分 — 429 是限速（你太快了），cf-mitigated 是滥用（它认为你是机器人）
x-desc: Cloudflare 有两个看起来像但修法完全不同的"出错"信号。**HTTP 429 是速率限制** —— 你的请求太快，遵守 `Retry-After` 降速后重试即可。**`cf-mitigated` 是滥用检测** —— Cloudflare 怀疑你是机器人，这不是速率限制，要换 UA / IP / 节奏，重试只会更糟。
x-sidebar: Cloudflare 429 vs cf-mitigated
x-keywords: cloudflare, ratelimit, qps, api 配额, workers, 免费层, cf-mitigated, 滥用检测, 429, retry-after
x-json-ld:
  '@context': https://schema.org
  '@graph':
    - '@type': TechArticle
      headline: 'Cloudflare 429 vs cf-mitigated — 它们不是一回事'
      inLanguage: 'zh-CN'
      about: 'Cloudflare 速率限制（429）和滥用检测（cf-mitigated），以及为何两者不同'
---

# Cloudflare 报错时怎么区分 — 429 速率限制 与 cf-mitigated 滥用检测 不是一回事

Cloudflare 有 **两个长得像但意思完全不同的"出错了"信号**。它们容易被
混淆 —— 而把一个搞错了修法只会让另一个更糟。

| 信号 | 是什么意思 | 怎么触发 |
| --- | --- | --- |
| **`HTTP 429 Too Many Requests`** | **速率限制。** 你在窗口内发请求太多了。遵守 `Retry-After`。 | 纯请求计数超出你 token 的配额。 |
| **`cf-mitigated: challenge` 或 `cf-mitigated: block`** | **滥用检测。** Cloudflare 觉得你的客户端看起来像机器人/自动化。**不是速率限制。** | WAF 启发式 —— User-Agent 字符串、自动化节奏、请求频率、IP 信誉。 |

本文接下来按各自单独讲：

1. **REST API 配额** —— 429 是从哪里来的。
2. **各产品的 HTTP 上限** —— 免费层各产品自己的限流。
3. **你请求太快了，Cloudflare 怎么处理** —— 429 的修法。
4. **Cloudflare 把你当成机器人** —— cf-mitigated 的修法（**不是速率限制**）。
5. **客户端遇到这两种情况怎么办** —— 一个 client 同时处理两种情况。

---

## REST API 配额 —— 429 是从哪里来的

大多数 Cloudflare REST API 端点共享一个按用户配额：

| 套餐 | 上限 | 窗口 | 每 |
| --- | --- | --- | --- |
| Free | 1200 次 | 5 分钟 | 用户 |
| Pro | 1200 次 | 5 分钟 | 用户 |
| Business | 1200 次 | 5 分钟 | 用户 |
| Enterprise | 自定义 | 自定义 | 用户 |

各套餐的每用户配额相同；差别在产品级 HTTP 上限（下一节）。Free
与 Pro 的"1200 次 / 5 分钟"是 Cloudflare 有意为之 —— 套餐差异在 API
的 *消费者侧*，不在 *操作者侧*。

### "每用户"在此处是什么意思

Cloudflare 的 API 速率限制以 **API token**（或 API key，
对旧集成）作为键。一个用户有两把 API token 就有两个独立的配
额 —— token 之间不共享。这是故意的：CI 任务可以各拿一把 token
以隔离突发。

### 单独配额的端点

少数端点有独立配额，而不是共用 1200/5min：

- `GET /zones/:id`（zone 详情）—— 较高，支撑 zone 列表类工作流
- DNS 读端点 —— 历史上较高，支撑批量枚举
- Workers KV / R2 / D1 —— 这些是 *产品级* 配额（见下），不是
  REST API 配额

新的端点例外会不定期添加；以 live 的
`developers.cloudflare.com/fundamentals/api/reference/limits/` 为准。

---

## 各产品的 HTTP 上限

| 产品 | Free | Pro | Business | Enterprise |
| --- | --- | --- | --- | --- |
| HTTP 请求 / zone / 天 | 100K | 10M / 月 | 100M / 月 | 自定义 |
| Workers 请求 / 天 | 100K | 1M / 月 | 20M / 月 | 自定义 |
| Pages 请求 | 无限（带宽上限） | 无限（带宽上限） | 无限（带宽上限） | 自定义 |
| KV 读 / 天 | 100K | 10M | 100M | 自定义 |
| KV 写 / 天 | 1K | 1M | 10M | 自定义 |
| KV 删 / 天 | 1K | 1M | 10M | 自定义 |
| R2 操作 / 月 | 10M（A 级） | 50M | 自定义 | 自定义 |
| D1 读 / 天 | 5M | 5B 行 / 月 | 自定义 | 自定义 |

这些上限与 REST API 配额**分开** —— Workers 配额用完不影响 API 配
额，反之亦然。两边都看。

---

## 你请求太快了，Cloudflare 怎么处理

当你超出按 token 的配额（默认 1200/5min）时，Cloudflare 响应：

```http
HTTP/1.1 429 Too Many Requests
Retry-After: 60
Content-Type: application/json
{"success": false, "errors": [{"code": 10000, "message": "Rate limit exceeded"}]}
```

**这是速率限制。** 修法：

1. **遵守 `Retry-After`** —— 响应告诉你什么时候回来。在那个时间
   之前别重试。
2. **降速** —— 你的请求率对该 token 的配额太高了。用指数退避
   加 jitter。
3. **本地跟踪配额** —— 别打到 429 之前就已经在跟踪本地的使用量。
4. **用多个 token** —— 每个 token 有自己的 1200/5min。把负载分散
   到 CI 任务。

`429` 是**可恢复**的。降速，配额重置，你继续。

---

## Cloudflare 把你当成机器人（不是速率限制，是滥用检测）

当 Cloudflare 的 WAF / 滥用检测觉得你的客户端像自动化、像可疑的或
像恶意的，它返回一个挑战页或拦截页，并设 `cf-mitigated` 头：

```http
HTTP/1.1 403 Forbidden
cf-mitigated: challenge
cf-ray: ...
```

或：

```http
HTTP/1.1 403 Forbidden
cf-mitigated: block
```

**这不是速率限制。** 这是 Cloudflare 的滥用防御 —— WAF 启发式标
记了你的客户端。常见的触发原因：

- **User-Agent 字符串** —— `python-requests/2.31.0`、
  `curl/8.4.0`、`Java/17.0.5` 等。真实浏览器发 `Mozilla/5.0 ...`。
- **自动化节奏** —— 完美的 1 秒间隔，没有人类式的变化。
- **单 IP 请求量** —— 持续高 QPS，特别是打到同一端点。
- **无头浏览器信号** —— 缺插件、缺字体、无 canvas / WebGL 指纹。
- **IP 信誉** —— 数据中心 IP、Tor 出口节点、有劣史的住宅代理。

修法**根本不同**于 429：

| 429（速率限制） | cf-mitigated（滥用） |
| --- | --- |
| 降速 | **降速 + 改客户端身份** |
| 遵守 `Retry-After` | 没 `Retry-After`；要改模式 |
| 同一客户端，更低速率 | 不同 UA、不同节奏、可能要不同 IP |
| 几秒就恢复 | 可能几小时不恢复，或要换 IP |

`cf-mitigated` **很难自动恢复**。客户端通常需要从根本上降速（人类
式节奏）、换 User-Agent、可能要换 IP。或换网络。

---

## 客户端遇到这两种情况怎么办

一个健壮的 Cloudflare 客户端分别处理两种情况：

```python
import time
import random

def call_cloudflare(url, token, max_retries=5):
    for attempt in range(max_retries):
        response = requests.get(
            url,
            headers={"Authorization": f"Bearer {token}"}
        )

        # 情况 1：速率限制 — 退避重试
        if response.status_code == 429:
            retry_after = int(response.headers.get("Retry-After", "60"))
            backoff = retry_after + random.uniform(0, 5)
            time.sleep(backoff)
            continue

        # 情况 2：滥用检测 — 升级处理
        if "cf-mitigated" in response.headers:
            log.warning(f"cf-mitigated: {response.headers['cf-mitigated']}")
            # 别自动重试 — 大幅退避或换身份
            raise AbuseDetected(
                f"Cloudflare WAF 触发: {response.headers['cf-mitigated']}"
            )

        # 成功或其他错误
        return response

    raise RateLimitExceeded()
```

三条经验：

1. **`429` 遵守 `Retry-After`**。别在那个时间之前重试。
2. **不要自动重试 `cf-mitigated`**。WAF 标记了你的客户端模式；
   用同模式重试就是一直被标记。大幅退避或换身份。
3. **`429` 用指数退避加 jitter**。裸指数退避会引发惊群效应，多个
   客户端同时打到上限时会同时重试。

---

## 本文不涵盖的

- **DDoS 防护**（与速率限制分开）。
- **Bot management**（Cloudflare 的 Bot Fight Mode / Super Bot
  Fight Mode / Bot Management for Enterprise）。
- **你自己在 zone 上设的速率限制规则**（这些是你域名上的配置，
  不是 Cloudflare 账户级限制）。
- **Cloudflare WAF 的 IP allow/block 列表**。

这些是不同层；各自看 Cloudflare 的文档。

---

## 来源

- REST API 每用户限流：
  <https://developers.cloudflare.com/fundamentals/api/reference/limits/>
- 按产品的 HTTP 上限：
  - Workers：<https://developers.cloudflare.com/workers/platform/limits/>
  - KV：<https://developers.cloudflare.com/kv/platform/limits/>
  - R2：<https://developers.cloudflare.com/r2/platform/limits/>
  - D1：<https://developers.cloudflare.com/d1/platform/limits/>
- Free / Pro / Business / Enterprise 套餐价格：
  <https://www.cloudflare.com/plans>
- `cf-mitigated`（WAF / 滥用检测语义）：
  <https://developers.cloudflare.com/fundamentals/reference/protections/>