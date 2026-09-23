---
x-title: GitHub 速率限制 —— 速查（5 种限速 + 5 种下载暴露面）
x-desc: 速查表：GitHub 5 种限速（REST 5000/小时、GraphQL 5000 点/小时、Search 30/分钟、Actions 1000/小时、二级启发式）+ 5 种下载暴露面（Releases API / HTML / archive / raw / CDN），其中只有 Releases API 计入 API 配额。
x-sidebar: GitHub 速率限制 + 替代方案
x-keywords: github, ratelimit, api 配额, rest api, graphql, actions, 二级速率限制, x-ratelimit, oauth, github app, releases api, codeload, raw.githubusercontent.com, jsdelivr
x-json-ld:
  '@context': https://schema.org
  '@graph':
    - '@type': TechArticle
      headline: 'GitHub 速率限制'
      inLanguage: 'zh-CN'
      about: 'GitHub API 速率限制跨 REST、GraphQL、Search、Actions 和二级'
---

# GitHub 速率限制 —— 速查（5 种限速 + 5 种下载暴露面）

---

## 速查

### 一、5 种速率限制一览

| 限速面 | 认证 | 上限 | 窗口 | 单位 |
| --- | --- | --- | --- | --- |
| **Primary REST** | PAT / OAuth / GitHub App | 5000 req | 1 小时 | token / installation |
| **Primary REST** | 无 | 60 req | 1 小时 | 源 IP |
| **GraphQL** | 任意 | 5000 点 | 1 小时 | token / installation（cost-based） |
| **Search** | 任意 | 30 req | 1 分钟 | 用户 |
| **Actions API** | 任意 | 1000 req | 1 小时 | 仓库 |
| **二级** | — | 启发式 | — | 滥用检测 |

**单位 key**：PAT / OAuth / user-to-server 是 **per token**（多 token = 多倍预算）；GitHub App 是 **per installation**（多 installation = 多倍聚合）；Search / Actions / GraphQL / REST core **分桶**，`X-RateLimit-Resource` header 告诉你当前桶。

### 二、5 种下载暴露面（**只有 Releases API 计入 API 配额**）

| 暴露面 | URL | 计入 API 配额？ | 适用 |
| --- | --- | --- | --- |
| **Releases API** | `api.github.com/...` | ✅ 是（5000/小时 auth） | 列出 release、找资产 URL |
| **HTML 抓取** | `github.com/.../releases/...` | ⚠️ 是（未文档化 UI 限流，~数百/小时/IP） | API 耗尽的一次性回退 |
| **Archive tarball** | `codeload.github.com/.../tar.gz/refs/...` 或 `github.com/.../archive/.../tar.gz` | ❌ 否（Fastly CDN） | 已知 ref 的整仓快照（≤100 MB） |
| **Raw 内容** | `raw.githubusercontent.com/...` | ❌ 否（Fastly CDN） | 按路径单文件抓取 |
| **CDN 镜像** | `cdn.jsdelivr.net/gh/...` / `cdn.statically.io/gh/...` / `gcore.jsdelivr.net/gh/...` | ❌ 否（CDN 级） | GitHub 慢 / 节流 / 不可达时的回退 |

**核心策略**：1 个 API 调用列出 release + CDN 拉资产 = API 预算 1 次，下载无限。

```sh
# 第 1 步：列出 release（1 个 API 调用）
curl -H "Authorization: Bearer $TOKEN" \
     'https://api.github.com/repos/x-cmd/x-cmd/releases?per_page=100'

# 第 2 步：下载（CDN，无 API 成本）
curl -L -o release.tar.gz \
     https://github.com/x-cmd/x-cmd/archive/refs/tags/v1.0.0.tar.gz

# 第 3 步：单文件（CDN，无 API 成本）
curl -L -o README.md \
     https://raw.githubusercontent.com/x-cmd/x-cmd/v1.0.0/README.md
```

### 三、关键响应头

| Header | 含义 | 何时 |
| --- | --- | --- |
| `X-RateLimit-Limit` | 当前桶总额 | 每次响应 |
| `X-RateLimit-Remaining` | 剩余 | 每次响应 |
| `X-RateLimit-Reset` | **UNIX 时间戳**（不是"还剩 N 秒"） | 每次响应 |
| `X-RateLimit-Used` | 已用 | 每次响应 |
| `X-RateLimit-Resource` | 当前桶（`core` / `search` / `graphql` 等） | 每次响应 |
| `Retry-After` | 整数秒 | 429 时 |

**陷阱**：`X-RateLimit-Reset` 是 UNIX epoch（如 `1640000000`），不是相对值。`Retry-After` 才是相对秒数。新手常踩。

### 四、429 / 403 怎么响应

| 触发 | HTTP | 关键 header | 修法 |
| --- | --- | --- | --- |
| Primary 配额到 | 403 | `X-RateLimit-Remaining: 0` | 睡到 `X-RateLimit-Reset` |
| Secondary 启发式触发 | 429 | `Retry-After` | 遵守 `Retry-After` + 改模式 |
| Search 配额到 | 403 | `X-RateLimit-Resource: search` | 睡 1 分钟 + 少搜 |

**注意**：Primary 和 Secondary 的 429 看起来一样——单从响应分不出哪个桶触发的。

---

## 正文

### 一、Primary REST —— 5000/小时 token

PAT / OAuth / GitHub App user-to-server 都用这个桶（GitHub App 算 installation）。配额的 key 是 **token 或 installation**，不是 GitHub 账号——3 个 PAT = 3 个独立预算。

GitHub Apps 可[申请更高配额](https://docs.github.com/en/apps/creating-github-apps/setting-up-a-github-app/about-choosing-a-github-app)，但典型工作流 5000/小时够用。

### 二、GraphQL —— 5000 点/小时 cost-based

每个查询按**最高成本字段**扣 1-10 点：

```graphql
query {
  repository(name: "x-cmd", owner: "x-cmd") {
    issues(first: 10) {       # 1 点
      nodes {
        comments(first: 100)  # 10 点（最高）
      }
    }
  }
}
```

上述查询扣 **10 点**（任一字段的最高值）。Connection 字段和聚合字段比简单字段读取贵。

**查消耗**：query 里加 `rateLimit { cost remaining resetAt }` 字段直接读。

### 三、Search / Actions / Secondary

**Search** —— 30/分钟，独立桶。昂贵因为索引 + 排序。REST 主配额**不**覆盖 `/search/*`——分桶。

**Actions API** —— 1000/小时/repo，`/repos/<o>/<r>/actions/*` 下所有端点共用。重度轮询 Actions 的 dashboard 集成会撞。

**二级** —— 启发式滥用检测，无公开阈值，触发条件：
- 短时间内突发（即使主配额剩很多）
- 并发在途请求多
- 短时间内重复相同内容

触发时返回 429 + `Retry-After`，**跟 primary 429 看起来一样**。单从响应分不出哪个桶触发。

### 四、客户端怎么处理

```python
import time
import requests

def call_github(url, headers, max_retries=5):
    for attempt in range(max_retries):
        r = requests.get(url, headers=headers)
        if r.status_code == 200:
            return r
        if r.status_code == 429:
            # primary 或 secondary 都走 Retry-After
            time.sleep(int(r.headers.get("Retry-After", "60")))
            continue
        if r.status_code == 403:
            if r.headers.get("X-RateLimit-Remaining") == "0":
                # primary 配额到，睡到 reset
                wait = max(int(r.headers["X-RateLimit-Reset"]) - time.time(), 1)
                time.sleep(min(wait, 3600))
                continue
        r.raise_for_status()
    raise RateLimitExceeded()
```

要点：

1. **每次请求前查 `X-RateLimit-Remaining`**。0 就不发，睡到 reset。
2. **遵守 `Retry-After`**。Primary 和 secondary 都设。
3. **本地跟踪 per-token 预算**。`X-RateLimit-Remaining` 是权威的，但不要为查它发请求——本地存计数器，每次扣。
4. **避免突发**。并行 ≤ 5-10，否则 secondary 会触发。
5. **用 conditional request**。`If-None-Match` / `If-Modified-Since` 让 GitHub 返 304，不耗配额。
6. **尽量 CDN**。列 release 用 API（1 次），下载用 archive / raw / CDN（不耗）。

---

## 不在本文

- **GitHub Apps 高级配额申请流程** —— 见 [docs](https://docs.github.com/en/apps)。
- **GraphQL 字段级成本表** —— 见 [GraphQL resource limits](https://docs.github.com/en/graphql/overview/resource-limitations)。
- **`x eget` 实现的完整 mechanic** —— 见 FAQ `eget-comprehensive-considerations`。

---

## 来源

- Primary REST: <https://docs.github.com/en/rest/using-the-rest-api/rate-limits-for-the-rest-api>
- GraphQL: <https://docs.github.com/en/graphql/overview/resource-limitations>
- Search: <https://docs.github.com/en/rest/search>
- Actions: <https://docs.github.com/en/rest/actions>
- Secondary: <https://github.blog/developer-skills/github/how-to-prevent-secondary-rate-limit-issues/>
- GitHub Apps auth: <https://docs.github.com/en/apps/creating-github-apps/authenticating-with-a-github-app/about-authentication-with-a-github-app>