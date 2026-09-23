---
x-title: Cloudflare 速率限制 —— 速查（status × cf-mitigated 组合）
x-desc: Cloudflare 出错时怎么读 (status code, cf-mitigated header) 二元组：429 + Retry-After 是速率限制；403 + cf-mitigated 是 WAF / Bot 挡机器人。完整速查表 + 各产品配额 + 客户端处理。
x-sidebar: Cloudflare 速率限制
x-keywords: cloudflare, ratelimit, qps, api 配额, workers, 免费套餐, cf-mitigated, retry-after, 429, 403
x-json-ld:
  '@context': https://schema.org
  '@graph':
    - '@type': TechArticle
      headline: 'Cloudflare 速率限制'
      inLanguage: 'zh-CN'
      about: 'Cloudflare API 与各产品速率限制'
---

# Cloudflare 速率限制 —— 速查（status × cf-mitigated 组合）

---

## 一、REST API 配额

| 套餐 | 配额 |
| --- | --- |
| Free | 1200 req / 5 min / API token |
| Pro | 1200 req / 5 min / API token |
| Business | 1200 req / 5 min / API token |
| Enterprise | 自定义 |

**单位是 API token，不是 Cloudflare 账号**：

- **API token** 是你在 [dash.cloudflare.com/profile/api-tokens](https://dash.cloudflare.com/profile/api-tokens) 里 Generate 出来的——大部分脚本和 curl 用的是这个。每次 Create Token 可以限定权限（"这把只能读 DNS，那把可以写 Workers"），可以撤销，可以只读。
- 一把 token 一个独立配额。两把 token = 两个配额池。
- CI 任务给每个 job 一把 token——一个写炸不会拖全队。
- Global API Key（旧式，2024 前集成）一把共享——CI 老炸是这个原因。

[来源：developers.cloudflare.com/fundamentals/api/reference/limits/](https://developers.cloudflare.com/fundamentals/api/reference/limits/)

---

## 二、各产品 HTTP 上限（和 REST API 配额分开计）

| 产品 | Free | Pro | Business |
| --- | --- | --- | --- |
| HTTP 请求 / zone / 天 | 100K | 10M / 月 | 100M / 月 |
| Workers 请求 / 天 | 100K | 1M / 月 | 20M / 月 |
| Pages 请求 | 无限（带宽上限） | 无限（带宽上限） | 无限（带宽上限） |
| KV 读 / 天 | 100K | 10M | 100M |
| KV 写 / 天 | 1K | 1M | 10M |
| KV 删 / 天 | 1K | 1M | 10M |
| R2 操作 / 月 | 10M（A 级） | 50M | 自定义 |
| D1 读 / 天 | 5M | 5B 行 / 月 | 自定义 |

Workers / KV / R2 / D1 按产品算——和 REST API 配额**完全分开**。Workers 用完不影响 API 配额。Enterprise 全部自定义——谈。

[来源：Workers](https://developers.cloudflare.com/workers/platform/limits/) · [KV](https://developers.cloudflare.com/kv/platform/limits/) · [R2](https://developers.cloudflare.com/r2/platform/limits/) · [D1](https://developers.cloudflare.com/d1/platform/limits/)

---

## 三、关键响应头

| Header | 含义 | 何时 |
| --- | --- | --- |
| `Retry-After` | 整数秒，等这么久再试 | 429 时 |
| `cf-mitigated` | 见第五节 | 各种 WAF / 配额场景 |
| `cf-ray` | CF 内部追踪 ID；找 CF 客服时给这个 | 任何错误 |
| `cf-cache-status` | 缓存命中情况 | 任何响应 |

CF **不像 GitHub**，没有 `X-RateLimit-*` / `X-RateLimit-Reset` 系列。**429 + `Retry-After` 是唯一可靠的限速信号**。

跨服务对比见 [2-rate-limit-headers-cheatsheet](2-rate-limit-headers-cheatsheet) §一。

---

## 四、客户端怎么处理

```python
import time
import random

def call_cloudflare(url, token, max_retries=5):
    for attempt in range(max_retries):
        r = requests.get(url, headers={"Authorization": f"Bearer {token}"})

        if r.status_code == 429:
            # 限速——遵守 Retry-After，退避重试
            wait = int(r.headers.get("Retry-After", "60"))
            time.sleep(wait + random.uniform(0, 5))
            continue

        if "cf-mitigated" in r.headers:
            # WAF / Bot——别自动重试，改模式或换 IP
            raise AbuseDetected(r.headers["cf-mitigated"])

        return r

    raise RateLimitExceeded()
```

要点：

1. **`429` 严格遵守 `Retry-After`**——在那个时间之前别重试。
2. **任何 `cf-mitigated` 都别自动重试**——WAF 在模式匹配，同模式重试只会一直被挡。
3. **`429` 用指数退避加 jitter**——避免惊群。
4. **多 token 隔离 CI job**——每把 token 是独立配额池。
5. **监控 `cf-mitigated` 比例**——比例突增 = 客户端模式或 IP 信誉变化，不是配额问题。

---

## 五、status × cf-mitigated —— 怎么读响应

| HTTP 状态 | `cf-mitigated` | 含义 | 修法 |
| --- | --- | --- | --- |
| `429` | （无） | **REST API 配额**（1200/5min/token） | 降速 + `Retry-After` |
| `429` | `rate-limit` | **域名层 rate limit 规则** | 降速 + `Retry-After` |
| `403` | `challenge` | **WAF challenge** | 解 challenge / 改模式 |
| `403` | `block` | **WAF 直接挡** | 改节奏 + UA / 换 IP |
| `403` | `bot` | **Bot Management 触发** | 同 `block` |
| `403` | `ip` | **IP 规则** | 换 IP |
| `403` | `country` | **国家规则** | 换 IP |
| `200` | `challenge` | **JS challenge 页**（HTML） | headless 跑 JS |

**每种响应是什么**：

- **`429` + （无）** —— 真·REST API 配额到了。详细修法见第四节。
- **`429` + `rate-limit`** —— 域名层 rate limit 规则触发，不是账户配额。
- **`403` + `challenge`** —— WAF 给了 challenge（CAPTCHA / JS）。解决 challenge，或改客户端模式。
- **`403` + `block`** —— WAF 直接挡，没有解决路径。必须改客户端模式或换 IP。
- **`403` + `bot`** —— Bot Management（Cloudflare 的反机器人产品，比 WAF 更激进）触发（要 Enterprise 或 Super Bot Fight Mode 才开）。
- **`403` + `ip` / `country`** —— IP / 国家规则触发。只能换 IP / 出口。
- **`200` + `challenge`** —— Bot Management 给的 JS challenge 页（HTML 含 JS）。headless 浏览器能跑，curl / requests 不行。

**为什么 Cloudflare 要拦你**：Cloudflare 不是给你找麻烦。**Cloudflare 在替它的客户（用 Cloudflare 的网站）挡可疑流量**。网站客户付钱是为了"我的站只服务真人"。WAF / Bot Management 是保护这个价值。你（调用方）被挡，是因为你的客户端模式撞到了 CF 给网站设的保护层。懂了这一层，"降速 + 换 UA + 换 IP" 不是绕弯，是 CF 设计上希望你这么干。

**触发条件**（启发式，CF 不公开具体阈值）：

- **User-Agent** —— `python-requests/2.31.0`、`curl/8.4.0` 等裸露信号；真实浏览器发 `Mozilla/5.0 ...`。
- **节奏** —— 完美 1 秒间隔，无人类式抖动。
- **单 IP QPS** —— 持续高 QPS 打到同一端点。
- **无头浏览器指纹** —— 缺插件、缺字体、canvas / WebGL 指纹异常。
- **IP 信誉** —— 数据中心 IP、Tor 出口、劣史住宅代理。

**`challenge` vs `block`**：

- **`challenge`** —— Cloudflare 给你个 CAPTCHA 或 JS challenge。**解决后通常能继续**。
- **`block`** —— Cloudflare 直接挡，没有解决路径。**必须改客户端模式或换 IP**。

**JS challenge 页怎么搞**：Cloudflare 返回 `200 OK` + 一个看起来正常的 HTML 页，里面嵌 JS challenge。客户端要执行 JS 算出 cookie（`cf_clearance`）让后续请求通过。Headless 浏览器（puppeteer / playwright）能跑——内置 JS 引擎。`curl` / `requests` 默认跑不了——没 JS 引擎。要么用 `cloudscraper` / `undetected-chromedriver`，要么放弃。

---

