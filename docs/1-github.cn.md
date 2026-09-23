---
x-title: GitHub 速率限制 —— 5000/h token + 二级启发式 + 5 种下载方式
x-desc: GitHub 5 种限速：认证 REST 5000/h token / Search 30/分钟 / Actions 1000/h/repo / GITHUB_TOKEN 1000/h/repo / 二级启发式。撞限速后 5 种替代下载：Releases API / HTML / archive / raw / CDN（只有 Releases API 计入 API 配额）。
x-sidebar: GitHub 速率限制
x-keywords: github, 速率限制, ratelimit, api 配额, rest api, graphql, actions, 二级速率限制, x-ratelimit, pat, github app, releases api, codeload, raw.githubusercontent.com, jsdelivr
x-json-ld:
  '@context': https://schema.org
  '@graph':
    - '@type': TechArticle
      headline: 'GitHub 速率限制'
      inLanguage: 'zh-CN'
      about: 'GitHub API 速率限制跨 REST、GraphQL、Search、Actions 和二级'
---

# GitHub 速率限制 —— 速查（5 种限速 + 5 种下载方式）

---

## 一、5 种速率限制

| 限速类型 | 认证 | 上限 | 窗口 |
| --- | --- | --- | --- |
| **Primary REST** | PAT（个人访问令牌）/ OAuth / GitHub App | 5000 req / token 或 installation | 1 小时 |
| **Primary REST** | 无 | 60 req / source IP | 1 小时 |
| **GitHub Actions 自动 token** | `${{ secrets.GITHUB_TOKEN }}` | 1000 req / repo（同一 repo 的所有 workflow run 共享）| 1 小时 |
| **GraphQL** | PAT / OAuth / GitHub App（任一即可） | 5000 点 / token（按查询成本算） | 1 小时 |
| **Search** | PAT / OAuth / GitHub App（任一即可） | 30 req / user | 1 分钟 |
| **Actions API** | PAT / OAuth / GitHub App（任一即可） | 1000 req / repo | 1 小时 |
| **二级** | — | 启发式触发 | — |

**配额按什么算**（看 "上限" 列的 `/ X` 部分）：
- `token`：每把 token 一份独立预算。3 把 PAT = 3 个独立 5000/h 桶。**PAT 是你在 GitHub 右上角头像 → Settings → Developer settings → Personal access tokens → Generate new token 里生成的那个**——大部分脚本和 curl 用的是这个。
- `installation`：每个 GitHub App 装一仓一份独立预算。GitHub App 是装在某个仓或组织的第三方应用（如 Dependabot、各种 CI 工具）。
- `repo`（GITHUB_TOKEN 时）：每个 repo 一份预算，**同 repo 的所有 workflow run 共享**——一个 workflow 写炸会拖累其他 workflow。
- `user`、`source IP`：按客户端标识分桶。
- REST、GraphQL、Search、Actions 是 4 个独立的桶（不互相挤占）。`X-RateLimit-Resource` header 告诉你当前在哪个桶。

**各限速类型 mechanic**：

- **Primary REST** —— PAT / OAuth / GitHub App 的用户令牌都用这个桶（GitHub App 按 installation 算）。配额按 token 或 installation 算，不是按 GitHub 账号——3 把 PAT = 3 个独立预算。GitHub Apps 可[申请更高配额](https://docs.github.com/en/apps/creating-github-apps/setting-up-a-github-app/about-choosing-a-github-app)，但典型工作流 5000/小时够用。

- **GraphQL** —— 每个查询按**最高成本字段**扣 1-10 点：

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

- **Search** —— 30/分钟，独立桶。搜索比 REST 主限速贵得多（每次都要重做索引 + 排序）。REST 主配额**不**覆盖 `/search/*`——是分开的桶。

- **Actions API** —— 1000/小时/repo，`/repos/<o>/<r>/actions/*` 下所有端点共用。重度轮询 Actions 的 dashboard 集成会撞。

- **二级** —— 启发式滥用检测，无公开阈值，触发条件：
  - 短时间内突发（即使主配额剩很多）
  - 并发在途请求多
  - 短时间内重复相同内容

  触发时返回 429 + `Retry-After`，**跟主限速 429 看起来一样**。单从响应分不出哪个桶触发。

---

## 二、撞上限制了？换种方式 -- 5 种替代下载

| 下载方式 | URL | 计入 API 配额？ | 适用 |
| --- | --- | --- | --- |
| **Releases API** | `api.github.com/...` | ✅ 是（5000/小时 auth） | 列出 release、找资产 URL |
| **HTML 抓取** | `github.com/.../releases/...` | ⚠️ 是（未文档化 UI 限流，~数百/小时/IP） | API 耗尽的一次性回退 |
| **Archive tarball** | `codeload.github.com/.../tar.gz/refs/...` 或 `github.com/.../archive/.../tar.gz` | ❌ 否（Fastly CDN） | 已知 ref 的整仓快照（≤100 MB） |
| **Raw 内容** | `raw.githubusercontent.com/...` | ❌ 否（Fastly CDN） | 按路径单文件抓取 |
| **jsDelivr CDN 镜像** | `cdn.jsdelivr.net/gh/...`（备用 `cdn.statically.io/gh/...` / `gcore.jsdelivr.net/gh/...`） | ❌ 否（CDN 级） | **撞限速 / GitHub 慢 / 不可达——首选** |

**核心策略**：用 API 列出 release（这是唯一扣配额的一步），下载走 CDN 或 raw。API 配额只扣 1 次，下载次数不限。

**首选 CDN = jsDelivr**——不只是 raw 镜像，还支持 semver（`@1` / `@^1.2`）、合并文件、npm 包、多 CDN 备份（`gcore.jsdelivr.net`）、免费额度 ~5000 万次/月。完整功能见 FAQ [`jsdelivr-github-capabilities`](#)。

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

---

## 三、关键响应头

| Header | 含义 | 何时 |
| --- | --- | --- |
| `X-RateLimit-Limit` | 当前桶总额 | 每次响应 |
| `X-RateLimit-Remaining` | 剩余 | 每次响应 |
| `X-RateLimit-Reset` | **UNIX 时间戳**（不是"还剩 N 秒"） | 每次响应 |
| `X-RateLimit-Used` | 已用 | 每次响应 |
| `X-RateLimit-Resource` | 当前桶（`core` / `search` / `graphql` 等） | 每次响应 |
| `Retry-After` | 整数秒 | 429 时 |

**陷阱**：`X-RateLimit-Reset` 是 UNIX 时间戳（像 `1640000000`），不是"还剩几秒"。要看剩余时间自己 `reset - now`。`Retry-After` 才是直接的"等几秒"，429 上才有。

---

## 四、429 / 403 怎么响应

| 触发 | HTTP | 关键 header | 修法 |
| --- | --- | --- | --- |
| Primary 配额到 | 403 | `X-RateLimit-Remaining: 0` | 睡到 `X-RateLimit-Reset` |
| Secondary 启发式触发 | 429 | `Retry-After` | 遵守 `Retry-After` + 改模式 |
| Search 配额到 | 403 | `X-RateLimit-Resource: search` | 睡 1 分钟 + 少搜 |

**注意**：主限速和二级的 429 看起来一样——单从响应分不出哪个桶触发的。

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
2. **遵守 `Retry-After`**。主限速和二级都设。
3. **本地跟踪每个 token 的预算**。`X-RateLimit-Remaining` 是权威的，但不要为查它发请求——本地存计数器，每次扣。
4. **避免突发**。并行 ≤ 5-10，否则二级会触发。
5. **用 conditional request**。`If-None-Match` / `If-Modified-Since` 让 GitHub 返 304，不耗配额。
6. **尽量 CDN**。列 release 用 API（1 次），下载用 archive / raw / CDN（不耗）。

---

## 五、GITHUB_TOKEN —— CI 场景的隐性撞墙点

`${{ secrets.GITHUB_TOKEN }}` 是 GitHub Actions 自动提供的 token：

- 每个 workflow run 自动创建、run 结束自动销毁。
- 默认开启，不用额外配置。
- **配额 1000 req/小时/repo**——所有 Actions API 端点共用这一个预算。

**CI 撞墙陷阱**：N 个 workflow 并发跑同一个 repo，全部用 GITHUB_TOKEN——它们**共享 1000/小时**。一个 workflow 写炸（大量轮询），其他 workflow 一起 429。

修法：
- **用 PAT 替代**——预算是 per token，每个 workflow 用一把 PAT 互不影响。
- **用 GitHub App 安装 token**——按 installation 隔离（每个仓一份独立配额）。
- **控制并发 + 用 conditional request**——见上文第四节要点。

---


---

## 来源

- Primary REST: <https://docs.github.com/en/rest/using-the-rest-api/rate-limits-for-the-rest-api>
- GraphQL: <https://docs.github.com/en/graphql/overview/resource-limitations>
- Search: <https://docs.github.com/en/rest/search>
- Actions: <https://docs.github.com/en/rest/actions>
- Secondary: <https://github.blog/developer-skills/github/how-to-prevent-secondary-rate-limit-issues/>
- GitHub Apps auth: <https://docs.github.com/en/apps/creating-github-apps/authenticating-with-a-github-app/about-authentication-with-a-github-app>