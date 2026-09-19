---
x-title: GitHub 速率限制 —— REST、GraphQL、Actions、二级
x-desc: GitHub 的主速率限制（认证 REST 5000/小时，未认证 60/小时，搜索 30/分钟，GraphQL 5000 点/小时），二级速率触发与响应头（X-RateLimit-*），以及健壮客户端的设计。
x-sidebar: GitHub 速率限制
x-keywords: github, ratelimit, api 配额, rest api, graphql, actions, 二级速率限制, x-ratelimit, oauth, github app
x-json-ld:
  '@context': https://schema.org
  '@graph':
    - '@type': TechArticle
      headline: 'GitHub 速率限制'
      inLanguage: 'zh-CN'
      about: 'GitHub API 速率限制：REST / GraphQL / Actions / 二级'
---

# GitHub 速率限制 —— REST、GraphQL、Actions、二级

GitHub 在主流 API 提供商中公开文档最齐全。四个面要追踪：

1. **主 REST** —— 认证 5000/小时，未认证 60/小时
2. **GraphQL** —— 5000 点/小时，按成本计
3. **搜索** —— 30/分钟，与 REST 主限分开
4. **二级** —— 启发式滥用检测速率限制

外加 **Actions API** 1000/小时/仓库，**GitHub App** 按安装
点配额。本文逐一走。

## 主 REST API 速率限制

| 认证面 | 上限 | 窗口 | 每 |
| --- | --- | --- | --- |
| 未认证 | 60 次 | 1 小时 | 源 IP |
| 个人访问令牌（PAT） | 5,000 次 | 1 小时 | token |
| OAuth app（用户令牌） | 5,000 次 | 1 小时 | token |
| GitHub App user-to-server | 5,000+ 次 | 1 小时 | 安装点 |
| GitHub App server-to-server | 5,000+ 次 | 1 小时 | 安装点 |

5,000/小时上限在 OAuth / PAT / user-to-server 之间相同。**GitHub
App 可以通过"increasing-api-quota-for-github-apps"表单申请更高配
额**，但典型工作流 5,000/小时足够。

### OAuth / PAT 的"per token"

配额是**每个 token**独立计算，不是每个用户。一个用户有三把 PAT
就有三个独立的 5,000/小时配额。CI 隔离场景下很有用：每个工作
流一把 PAT、各自独立预算。

### GitHub App：每个安装点配额

GitHub App 配额是**每个安装点**独立计算的 —— 5,000/小时是
按安装点计数，不是按 app 整体。如果你的 app 安装在 100 个仓
库的 100 个安装点上，你实际上累计能拿 500,000/小时，但每个
安装点的配额是独立追踪的。

## GraphQL API

GraphQL 用**成本制**配额：5,000 点/小时每个 token 或安装点。
每次查询消耗 1–10 点，取决于查询中**成本最高的字段**。

```graphql
query {
  repository(name: "x-cmd", owner: "x-cmd") {
    issues(first: 10) {       # 1 点
      nodes {
        comments(first: 100) # 10 点（max）
      }
    }
  }
}
```

上述查询成本 10 点（任意字段的最大值）。连接字段与聚合字段
成本高于简单字段读。

**小贴士**：在查询里加 `cost { totalCost }` 查看实际消耗：

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

## 搜索 API

`/search/*` 端点有独立、较低的配额：

| 面 | 上限 | 窗口 | 每 |
| --- | --- | --- | --- |
| 搜索 API（任何认证） | 30 次 | 1 分钟 | 用户 |

30/分钟远低于 REST 主限，因为搜索很贵（索引、排序）。REST
主限的 5,000/小时**不涵盖**`/search/*` —— 是独立预算。

## Actions API

| 面 | 上限 | 窗口 | 每 |
| --- | --- | --- | --- |
| REST API under `/repos/{owner}/{repo}/actions/*` | 1,000 次 | 1 小时 | 仓库 |

工作流 artifact 下载、list-runs、其它 Actions 相关 REST 端点
共用这个 1,000/小时/仓库配额。重度轮询 Actions 的 CI 工具（比
如 dashboard 集成）会撞上限。

## 二级速率限制（那个惊喜）

GitHub 在主限之上还跑一层**二级**速率限制，是启发式的——
由滥用模式触发：

- 短时间内突发请求过多（不管主限还剩多少）
- 在飞的并发请求
- 短时间内重复请求同样内容

触发时 GitHub 返回：

```http
HTTP/1.1 429 Too Many Requests
Retry-After: 60
X-RateLimit-Reset: 1640000000
```

没有公开的"二级剩余请求数"响应头。触发条件见
[GitHub 博客](https://github.blog/developer-skills/github/how-to-prevent-secondary-rate-limit-issues/)，
但具体阈值不公开。

### 避免二级限流

- **避免突发。** 即使主限还有 4,000/小时剩余，并行发 100 个
  请求也可能触发二级。
- **条件请求。** 用 `If-None-Match` / `If-Modified-Since`
  头 —— GitHub 对未变更资源返回 304，不消耗 API 配额。
- **避免紧轮询。** 不要每秒轮询；指数退避。

## GitHub 响应头

REST + GraphQL 标准响应头：

```http
X-RateLimit-Limit: 5000
X-RateLimit-Remaining: 4999
X-RateLimit-Reset: 1640000000
X-RateLimit-Used: 1
X-RateLimit-Resource: core    # "core", "search", "graphql", 等
Retry-After: 60                # 仅 429 时
```

**`X-RateLimit-Reset` 是 UNIX epoch 时间戳**，不是距离重置的
秒数。第一次写的客户端经常被这个坑到。

`X-RateLimit-Resource` 告诉你被追踪在哪个 bucket 里 —— 当单客
户端同时用 `/search/*`（30/分钟）和 core REST（5,000/小时）时这
很重要。

## 客户端重试策略

健壮的 GitHub 客户端：

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
            # 二级速率限制；准确 sleep Retry-After。
            time.sleep(retry_after)
            continue
        if response.status_code == 403:
            # X-RateLimit-Remaining == 0 意味着配额用尽。
            if response.headers.get("X-RateLimit-Remaining") == "0":
                reset_at = int(response.headers["X-RateLimit-Reset"])
                wait = max(reset_at - time.time(), 1)
                time.sleep(min(wait, 3600))
                continue
        # 非速率错误；让调用方处理。
        response.raise_for_status()
    raise RateLimitExceeded()
```

三条经验：

1. **每次请求前预查 `X-RateLimit-Remaining`。** 如果是 0 别
   发 —— 等到重置时间。
2. **出现 `Retry-After` 时遵守它。** 二级速率限制也会设这个
   头；主限在 429 上也设。
3. **本地按 token 跟踪预算。** GitHub 的 `X-RateLimit-
   Remaining` 是权威但你不必为查它发请求 —— 本地记数器
   减 1 即可，省掉"启动时我的配额是多少？"这个引导难题。

## 来源

- 主 REST 速率限制：
  <https://docs.github.com/en/rest/using-the-rest-api/rate-limits-for-the-rest-api>
- GraphQL 资源限制：
  <https://docs.github.com/en/graphql/overview/resource-limitations>
- 搜索 API 速率限制：
  <https://docs.github.com/en/rest/search>
- Actions API：
  <https://docs.github.com/en/rest/actions>
- 二级速率限制触发模式（GitHub 博客）：
  <https://github.blog/developer-skills/github/how-to-prevent-secondary-rate-limit-issues/>
- GitHub App 认证模型：
  <https://docs.github.com/en/apps/creating-github-apps/authenticating-with-a-github-app/about-authentication-with-a-github-app>

**核实状态**：2024-11 与上述来源比对核实。
