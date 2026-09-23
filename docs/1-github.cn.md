---
x-title: GitHub —— 速率限制以及如何用替代方案避免受限
x-desc: GitHub 的主速率限制（认证 REST 5000/小时，未认证 60/小时，搜索 30/分钟，GraphQL 5000 点/小时，Actions 1000/小时），二级速率触发与响应头（X-RateLimit-*）。文末附 4 行速查表，对比 5 种下载暴露面（Releases API / HTML / archive tarball / raw / CDN）以及哪些计入 API 配额。
x-sidebar: GitHub 速率限制 + 替代方案
x-keywords: github, ratelimit, api 配额, rest api, graphql, actions, 二级速率限制, x-ratelimit, oauth, github app, releases api, codeload, raw.githubusercontent.com, jsdelivr
x-json-ld:
  '@context': https://schema.org
  '@graph':
    - '@type': TechArticle
      headline: 'GitHub 速率限制'
      inLanguage: 'cn'
      about: 'GitHub API 速率限制跨 REST、GraphQL、Search、Actions 和二级'
---

# GitHub 速率限制

GitHub 是文档化最好的速率限制故事之一，跨主流 API 提供商。五个面
重要：

1. **主 REST** —— 5000/小时 认证，60/小时 未认证。
2. **GraphQL** —— 5000 点/小时，cost-based。
3. **Search** —— 30/分钟，与 REST 主分开。
4. **Actions API** —— 1000/小时/repo。
5. **二级** —— 启发式滥用检测速率限制。

外加五种**下载暴露面**（Releases API、HTML 抓取、archive tarball、
raw 内容、CDN 镜像）—— 只有一个计入 GitHub API 预算。文末速查表，
FAQ 里有完整策略与取舍（eget 策略、gitattributes、云开发环境）。

---

## 主 REST API 速率限制

| 认证面 | 限制 | 单次 | 每 |
| --- | --- | --- | --- |
| 未认证 | 60 req | 1 小时 | 源 IP |
| 个人访问 token（PAT） | 5,000 req | 1 小时 | token |
| OAuth app（用户 token） | 5,000 req | 1 小时 | token |
| GitHub App user-to-server | 5,000+ req | 1 小时 | installation |
| GitHub App server-to-server | 5,000+ req | 1 小时 | installation |

5,000/小时上限在 OAuth / PAT / user-to-server 间相同。**GitHub Apps
可通过 `increasing-api-quota-for-github-apps` 申请表请求更高配额**，
但对典型工作流 5,000/小时够了。

### "Per token" 对 OAuth / PAT 的含义

配额是 **per token** 而非 per user。一个有 3 个 PAT 的用户拿到 3 个独立
的 5,000/小时 配额。这对 CI 隔离有用：每个工作流一个 PAT，每个工作流一个
预算。

### GitHub Apps：per-installation 配额

GitHub Apps 有 **per-installation** 配额 —— 5,000/小时计入 installation
而非 app 整体。如果你的 app 安装在 100 个 installation 上，你实际上最多
拿 500,000/小时 聚合，但每个 installation 的配额独立追踪。

## GraphQL API

GraphQL 用 **cost-based** 配额：5,000 点/小时 per token-or-installation。
每个查询按查询中*最高成本字段*扣 1-10 点。

```graphql
query {
  repository(name: "x-cmd", owner: "x-cmd") {
    issues(first: 10) {       # costs 1 point
      nodes {
        comments(first: 100) # costs 10 points (max)
      }
    }
  }
}
```

上述查询扣 10 点（任一字段的最大值）。Connection 字段和聚合字段
比简单字段读取贵。

**Pro tip**：在查询里问 `cost { totalCost }` 检查实际消耗：

```graphql
query {
  repository(...) { ... }
  rateLimit {
    limit
    cost
    remaining
    resetAt
  }
}
```

## Search API

`/search/*` 端点有独立、更低的配额：

| 暴露面 | 限制 | 单次 | 每 |
| --- | --- | --- | --- |
| Search API（任何认证） | 30 req | 1 分钟 | 用户 |

30/分钟 远低于 REST 主限制，因为搜索昂贵（索引、排序）。REST 主
5,000/小时 配额**不**覆盖 `/search/*` —— 它们是分开的预算。

## Actions API

| 暴露面 | 限制 | 单次 | 每 |
| --- | --- | --- | --- |
| REST API 在 `/repos/{owner}/{repo}/actions/*` 下 | 1,000 req | 1 小时 | 仓库 |

Workflow 制品下载、list-runs 与其他 Actions 相关 REST 端点共用这个
1,000/小时/repo 配额。重度轮询 Actions 的 CI 工具（例如 dashboard 集成）
可能撞到这个。

## 二级速率限制（出其不意的那种）

GitHub 在主限制之上强制 *secondary* 速率限制。它是启发式的 —— 由滥用
样式触发：

- 短时间内过多请求（无论主配额剩余）。
- 并发在途请求。
- 短时间内重复请求相同内容。

触发时 GitHub 返回：

```http
HTTP/1.1 429 Too Many Requests
Retry-After: 60
X-RateLimit-Reset: 1640000000
```

没有"你还有 N 个 secondary 请求"的文档化头。触发条件在
[GitHub blog post](https://github.blog/developer-skills/github/how-to-prevent-secondary-rate-limit-issues/) 描述，但精确阈值不公开。

### 避免二级限制

- **避免突发。** 即使主配额剩 4,000/小时，并行发 100 个请求也能触发
  secondary。
- **Conditional 请求。** 用 `If-None-Match` / `If-Modified-Since` 头 ——
  GitHub 对未变资源返回 304，不消耗 API 配额。
- **避免紧密轮询。** 别每秒轮询；指数退避。

## GitHub 发的响应头

REST + GraphQL 上的标准头：

```http
X-RateLimit-Limit: 5000
X-RateLimit-Remaining: 4999
X-RateLimit-Reset: 1640000000
X-RateLimit-Used: 1
X-RateLimit-Resource: core    # "core"、"search"、"graphql" 等
Retry-After: 60                # 仅在 429 上
```

**`X-RateLimit-Reset` 是 UNIX 时间戳**，不是秒数。这是初次使用者的陷阱。

`X-RateLimit-Resource` 让你知道被追踪在哪个桶上 —— 重要，当一个客户端同时用
`/search/*`（30/分钟）和核心 REST（5,000/小时）时。

## 客户端重试策略

一个健壮的 GitHub 客户端：

```python
import time
import requests

def call_github(url, headers, max_retries=5):
    for attempt in range(max_retries):
        response = requests.get(url, headers=headers)
        if response.status_code == 200:
            return response
        if response.status_code == 429:
            retry_after = int(response.headers.get("Retry-After", "60"))
            time.sleep(retry_after)
            continue
        if response.status_code == 403:
            if response.headers.get("X-RateLimit-Remaining") == "0":
                reset_at = int(response.headers["X-RateLimit-Reset"])
                wait = max(reset_at - time.time(), 1)
                time.sleep(min(wait, 3600))
                continue
        response.raise_for_status()
    raise RateLimitExceeded()
```

三条经验：

1. **每次请求前预检 `X-RateLimit-Remaining`。** 如果是 0，别费心 ——
   睡到 reset。
2. **存在 `Retry-After` 时尊重它。** Secondary 速率限制会设它；primary
   在 429 上也会设。
3. **本地跟踪 per-token 预算。** `X-RateLimit-Remaining` 是权威的，但你
   不需要为查它发请求 —— 本地存计数器，每次请求扣一次。

---

## 下载暴露面 —— 速查表

同一个 `x-cmd/x-cmd` release 制品可以通过 **五个不同的 URL** 获取，
每种有自己的速率限制故事。完整策略与取舍在 FAQ（`github-five-
download-surfaces`）。

| 暴露面 | URL | 是否计入 API 预算？ | 适用 |
| --- | --- | --- | --- |
| **Releases API** | `api.github.com/...` | ✅ 是（5000/小时 认证 或 60/小时 未认证） | 列出 release、找资产 URL |
| **HTML 抓取** | `github.com/.../releases/...` | ⚠️ 是 —— 未文档化的 UI 限制，每 IP ~数百/小时 | API 耗尽时的一次性回退 |
| **Archive tarball** | `codeload.github.com/.../tar.gz/refs/...` 或 `github.com/.../archive/.../tar.gz` | ❌ 否（Fastly CDN） | 已知 ref 的整个仓库快照（≤100 MB 上限） |
| **Raw 内容** | `raw.githubusercontent.com/...` | ❌ 否（Fastly CDN） | 按路径单文件抓取 |
| **CDN 镜像** | `cdn.jsdelivr.net/gh/...`、`cdn.statically.io/gh/...`、`gcore.jsdelivr.net/gh/...` | ❌ 否（CDN 级别） | GitHub 慢 / 节流 / 不可用时的回退 |

**关键洞察**：只有 Releases API 计入。用 API 列出（1 个请求发现所有
release），然后用 `codeload.github.com` 或 `raw.githubusercontent.com`
下载 —— 都是 CDN 缓存，免于 API 预算。

**策略速查** —— 对于想获取 `x-cmd/x-cmd` 每个 release 的工具：

```sh
# 第 1 步：列出 releases（1 个 API 调用）
curl -H "Authorization: Bearer $TOKEN" \
     'https://api.github.com/repos/x-cmd/x-cmd/releases?per_page=100'

# 第 2 步：下载每个 release（CDN，无 API 成本）
curl -L -o release.tar.gz \
     https://github.com/x-cmd/x-cmd/archive/refs/tags/v1.0.0.tar.gz

# 第 3 步：抓取单文件（CDN，无 API 成本）
curl -L -o README.md \
     https://raw.githubusercontent.com/x-cmd/x-cmd/v1.0.0/README.md
```

API 预算：1 个请求。CDN 下载：无限。

---

## 源码

- 主 REST 速率限制：
  <https://docs.github.com/en/rest/using-the-rest-api/rate-limits-for-the-rest-api>
- GraphQL 资源限制：
  <https://docs.github.com/en/graphql/overview/resource-limitations>
- Search API 速率限制：
  <https://docs.github.com/en/rest/search>
- Actions API：
  <https://docs.github.com/en/rest/actions>
- Secondary 速率限制博客文章（触发模式）：
  <https://github.blog/developer-skills/github/how-to-prevent-secondary-rate-limit-issues/>
- GitHub Apps 认证模型：
  <https://docs.github.com/en/apps/creating-github-apps/authenticating-with-a-github-app/about-authentication-with-a-github-app>

**验证状态**：截至 2024-11 验证以上内容。