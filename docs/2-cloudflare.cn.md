---
x-title: Cloudflare 报错时怎么区分 — 429 是限速（你太快了），cf-mitigated 是滥用（它认为你是机器人）
x-desc: Cloudflare 有两个看起来像但修法完全不同的"出错"信号。**HTTP 429 是速率限制** —— 你的请求太快，遵守 `Retry-After` 降速后重试即可。**`cf-mitigated` 是滥用检测** —— Cloudflare 怀疑你是机器人，这不是速率限制，要换 UA / IP / 节奏，重试只会更糟。
x-sidebar: Cloudflare 429 vs cf-mitigated
x-keywords: cloudflare, ratelimit, qps, api 配额, workers, 免费套餐, cf-mitigated, retry-after, 429
x-json-ld:
  '@context': https://schema.org
  '@graph':
    - '@type': TechArticle
      headline: 'Cloudflare 速率限制'
      inLanguage: 'zh-CN'
      about: 'Cloudflare API 与各产品速率限制'
---

# Cloudflare 速率限制 —— REST API 与各产品配额

Cloudflare 的速率限制故事有两条独立的线，经常被搞混：

1. **REST API 配额** —— 每用户对 `api.cloudflare.com` 的
   请求预算。影响所有调 Cloudflare API 的人。
2. **各产品 HTTP 上限** —— zone 总 HTTP 请求、Workers 调用、
   KV 操作等。影响这些产品的客户。

还有第三件看起来像速率限制但其实不是的事：**`cf-mitigated`
挑战头**。在本文末尾单独讲。

## REST API 配额

大部分 Cloudflare REST API 端点共用一个每用户配额：

| 套餐 | 上限 | 窗口 | 每 |
| --- | --- | --- | --- |
| Free | 1200 次 | 5 分钟 | 用户 |
| Pro | 1200 次 | 5 分钟 | 用户 |
| Business | 1200 次 | 5 分钟 | 用户 |
| Enterprise | 自定义 | 自定义 | 用户 |

各套餐的每用户配额相同；差别在产品级 HTTP 上限（下一节）。
Free 与 Pro 的"1200 次 / 5 分钟"是 Cloudflare 有意为之 —— 套餐
差异在 API 的 *消费者侧*，不在 *操作者侧*。

### "每用户"在此处是什么意思

Cloudflare 的 API 速率限制以 **API token**（或 API key，
对旧集成）作为键。一个用户有两把 API token 就有两个独立的配
额 —— token 之间不共享。这是故意的：CI 任务可以各拿一把 token
以隔离突发。

### 单独配额的端点

少数端点有独立配额，而不是共用 1200/5min：

- `GET /zones/:id`（zone 详情）—— 较高，支撑 zone 列表类
  工作流
- DNS 读端点 —— 历史上较高，支撑批量枚举
- Workers KV / R2 / D1 —— 这些是 *产品级* 配额（见下），不是
  REST API 配额

新的端点例外会不定期添加；以 live 的
`developers.cloudflare.com/fundamentals/api/reference/limits/` 
为准。

## 各产品 HTTP 上限

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

## `cf-mitigated` 与 429 的区别

Cloudflare 有两种不同的响应信号：

- **`429 Too Many Requests`** —— 你的代码调用了太多 API 请求。
  遵守 `Retry-After` 并退避。
- **`cf-mitigated: challenge` 或 `cf-mitigated: block`** ——
  Cloudflare 检测到你客户端有滥用模式（User-Agent 字符串、自动
  化模式、请求频率等），返回了挑战或拦截页。**这不是速率限
  制**，是滥用防御。

`cf-mitigated` 的情况更难自动恢复 —— 客户端通常需要根本性地减
慢节奏（接近真人操作）或换网络。429 只是"慢一点"。

### 实用建议

```python
# 伪代码：区分两者
response = requests.get(...)
if response.status_code == 429:
    sleep(int(response.headers.get("Retry-After", "60")))
    retry()
elif response.headers.get("cf-mitigated"):
    # 真正的滥用缓解。不要立即重试；大幅退避或换身份。
    log.warning(f"cf-mitigated: {response.headers['cf-mitigated']}")
    raise AbusiveRequestDetected(...)
```

## 客户端重试策略

一个健壮的 Cloudflare 客户端：

```python
import time
import random

def call_cloudflare(url, token, max_retries=5):
    for attempt in range(max_retries):
        response = requests.get(url, headers={"Authorization": f"Bearer {token}"})
        if response.status_code != 429:
            return response
        # 指数退避 + jitter
        backoff = (2 ** attempt) + random.uniform(0, 1)
        time.sleep(min(backoff, 60))  # 最多 60 秒
    raise RateLimitExceeded()
```

三条经验：

1. **遵守 `Retry-After`** —— Cloudflare 在 429 上设置这个头，遵
   守它能避免立刻又撞上限。
2. **指数退避 + jitter** —— 纯指数退避在多个客户端同时撞上限时
   会引起雷暴群重试。
3. **每 token 单独预算跟踪** —— Cloudflare 给你 1200/5min，
   但你的 token 可能在多个进程间共享。本地跟踪用量以避免 429。

## 本文不涵盖的

- DDoS 防护（独立于速率限制）
- Bot 防御（Bot Fight Mode / Super Bot Fight Mode / Bot
  Management for Enterprise）
- 你自己在自己 zone 上设的速率限制规则（这是你域名的配置，
  不是 Cloudflare 账户级）

这些都是不同的话题；见 Cloudflare 对应文档。

## 来源

- REST API 每用户限流：
  <https://developers.cloudflare.com/fundamentals/api/reference/limits/>
- 各产品 HTTP 配额：
  - Workers：<https://developers.cloudflare.com/workers/platform/limits/>
  - KV：<https://developers.cloudflare.com/kv/platform/limits/>
  - R2：<https://developers.cloudflare.com/r2/platform/limits/>
  - D1：<https://developers.cloudflare.com/d1/platform/limits/>
- Free / Pro / Business / Enterprise 套餐价格：
  <https://www.cloudflare.com/plans>
- `cf-mitigated`（WAF / 滥用检测语义）：
  <https://developers.cloudflare.com/fundamentals/reference/protections/>
