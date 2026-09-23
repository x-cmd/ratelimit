---
x-title: GitHub —— 速率限制与下载策略（releases、HTML、archive、raw、CDN 镜像）
x-desc: GitHub 的主速率限制（认证 REST 5000/小时，未认证 60/小时，搜索 30/分钟，GraphQL 5000 点/小时，Actions 1000/小时），二级速率触发，外加五种下载策略深度对比：Releases API、HTML 抓取、archive/tarball、raw 内容、CDN 镜像 —— 以及它们与速率限制的交互。
x-sidebar: GitHub 速率限制 + 下载
x-keywords: github, ratelimit, api 配额, rest api, graphql, actions, 二级速率限制, x-ratelimit, oauth, github app, releases api, codeload, raw.githubusercontent.com, jsdelivr, gitattributes, sparse-checkout, colab, codespaces
x-json-ld:
  '@context': https://schema.org
  '@graph':
    - '@type': TechArticle
      headline: 'GitHub 速率限制与下载策略'
      inLanguage: 'cn'
      about: 'GitHub API 速率限制与五种下载策略'
---

# GitHub —— 速率限制与下载策略

本文两个故事：

1. **GitHub API 速率限制** —— 主 REST、GraphQL、Search、Actions、二级。
2. **GitHub 下载策略** —— 五种获取代码/发布制品的方法，以及每种与速率限制的交互。

第二个故事是让开发者措手不及的：**GitHub 通过五种不同的暴露面提供相同的内容**，
每种面有自己的速率限制语义。如果你一直撞 60 req/hr（未认证配额）而困惑 —— 可能你
用的是 API，而正确的面是 **raw.githubusercontent.com**（无速率限制）或 **archive
tarball**（CDN 缓存，无速率限制）。

---

## 为什么是 GitHub？

GitHub 是几乎每个开发者工具的上游来源。三种流量模式重要：

- **浏览** —— `github.com/<owner>/<repo>` —— UI 速率限制，未文档化但有
  强制（~数百/小时）。
- **API** —— `api.github.com/...` —— 已文档化的主速率限制（认证 5,000/小时）。
- **Raw / archive / CDN** —— `raw.githubusercontent.com` 与 `codeload.github.com` ——
  CDN 缓存，**无文档化的速率限制**（软限制适用）。

如果你大规模抓取 GitHub，选对暴露面比加 token 更重要。

---

## GitHub 下载策略 —— 五种获取方式

同一个 `x-cmd/x-cmd` 的 release 制品可以通过 **五个不同的 URL** 获取，
每种有自己的速率限制故事。

### 暴露面 1 —— Releases API（`api.github.com`）

程序化列出 release 与下载 release 制品的标准方式。

```sh
# 列出 releases
curl -H "Accept: application/vnd.github+json" \
     https://api.github.com/repos/x-cmd/x-cmd/releases

# 下载特定 release 的制品
curl -L \
     -H "Authorization: Bearer $TOKEN" \
     -o x-cmd.tar.gz \
     https://api.github.com/repos/x-cmd/x-cmd/releases/tags/v1.0.0
```

| 属性 | 值 |
| --- | --- |
| 速率限制 | **5000/小时 认证，60/小时 未认证**（GitHub 主 REST） |
| 限制 | 列出的 releases（默认 30）但分页 |
| 适用 | "当前版本是什么？"——列出、过滤、找制品 URL |
| 代价 | 每次调用扣 1 个 REST 配额 |

**限制**：

- **60 req/hr 未认证** —— 最常被引用的"GitHub 在限制我"抱怨。通常
  正确答案是加 token；有时正确答案是用别的暴露面（见下）。
- **二级速率限制** —— 突发模式即使有剩余配额也会触发启发式节流。
  见下文"二级速率限制"段。

### 暴露面 2 —— HTML 抓取（`github.com/.../releases/tag/...`）

当你没有 token 时，显而易见的回退是抓取 HTML release 页。

```sh
curl -L -A "Mozilla/5.0" \
     https://github.com/x-cmd/x-cmd/releases/tag/v1.0.0 \
     | grep -oP 'href="[^"]*\.(tar\.gz|zip)"'
```

| 属性 | 值 |
| --- | --- |
| 速率限制 | **未文档化的 UI 限制**（每个 IP ~数百/小时） |
| 限制 | 脆弱 —— 页面结构变化会无声破坏抓取器 |
| 适用 | 一次性手动下载；API 配额耗尽时的回退 |
| 代价 | 服务端速率限制；无 per-token 预算 |

**限制**：

- **未文档化阈值** —— GitHub 不公开数字。长期抓取会被 429。
- **无个人预算** —— 限制是 per-IP 而非 per-token。在 NAT 后面，
  整个办公室共享一个桶。
- **DOM 漂移** —— 下次 CSS class 重命名会破坏你的 regex。
- **无头检测** —— GitHub 给"浏览器式" User-Agent 与 `curl` UA
  提供不同 HTML。设置 UA 然后赌一把。

只在 API 配额真的耗尽且需要一次性拉取时用 HTML 抓取。生产环境
不用。

### 暴露面 3 —— Archive / tarball（仓库 ref 快照）

GitHub 速率限制最大的逃生口：**把整个仓库下载为 tarball**。Fastly CDN 提供，
不走 GitHub API。

```sh
# 分支 tarball
curl -L -o x-cmd-main.tar.gz \
     https://codeload.github.com/x-cmd/x-cmd/tar.gz/refs/heads/main

# 标签 tarball
curl -L -o x-cmd-v1.0.0.tar.gz \
     https://codeload.github.com/x-cmd/x-cmd/tar.gz/refs/tags/v1.0.0

# 或用 github.com URL（同一 CDN）
curl -L -o x-cmd-v1.0.0.tar.gz \
     https://github.com/x-cmd/x-cmd/archive/refs/tags/v1.0.0.tar.gz
```

| 属性 | 值 |
| --- | --- |
| 速率限制 | **仅 CDN（Fastly）；无文档化的 GitHub API 速率限制** |
| 大小上限 | 每个 archive 压缩后 ~100 MB（GitHub 硬性上限） |
| 适用 | **获取已知 ref 的整个仓库源码** |
| 代价 | 网络带宽，非 API 配额 |

**限制**：

- **仓库大小上限** —— GitHub 对 archive 强制 100 MB 压缩上限。更大
  的仓库无法作为单个 tarball 下载。改用 git clone。
- **LFS 文件是指针不是 blob。** Archive tarball 不包含真实 LFS 内容。
  单独下载 LFS。
- **Submodules** —— Archive tarball 不包含 submodules。对于带
  submodules 的仓库，git clone 是唯一完整保真选项。

这是**拯救大多数机器人**的暴露面 —— release 列表 API 配额被列出所有
release 耗尽，但 tarball 是单个 CDN 请求，无 API 预算成本。

### 暴露面 4 —— Raw 内容（`raw.githubusercontent.com`）

最简单的逃生口：**通过 GitHub 的 CDN 一次拿一个文件**。无 API、无 HTML 抓取、
无 tarball 展开。

```sh
# 单文件
curl -L https://raw.githubusercontent.com/x-cmd/x-cmd/main/README.md

# 指定标签
curl -L https://raw.githubusercontent.com/x-cmd/x-cmd/v1.0.0/INSTALL.md

# 子目录树（按文件用 raw URL，不用递归）
for f in install.d/01.sh install.d/02.sh install.d/03.sh; do
  curl -L "https://raw.githubusercontent.com/x-cmd/x-cmd/main/$f"
done
```

| 属性 | 值 |
| --- | --- |
| 速率限制 | **仅 CDN（Fastly）；无文档化的 GitHub API 速率限制** |
| 软限制 | 每个源 IP ~60-100 req/分钟（观察值；未公开） |
| 适用 | **已知路径的文件** —— manifests、配置、单脚本 |
| 代价 | 网络带宽，非 API 配额 |

**为什么 `raw.xxx` 绕过速率限制**：GitHub 的 API 速率限制适用于
`api.github.com` 与 `github.com` HTML 页面。`raw.githubusercontent.com` 是另一
条由 Fastly 提供的 CDN —— 与 `cdn.jsdelivr.net` 和 GitHub Releases 下载共用。你
没有消耗 API 预算。

**限制**：

- **软 IP-based 速率限制** —— 是有的，只是未文档化。重度自动化
  （例如，从一个 IP 每分钟拉几千个文件）会被节流。观察到的阈值是
  ~60-100 req/分钟。
- **无递归目录列举** —— `raw.githubusercontent.com` 不枚举；你需要知道
  路径。要递归下载，用 `git clone` 或 tarball。
- **无 auth** —— token 无用。`raw.xxx` 匿名设计。

### 暴露面 5 —— CDN 镜像（jsDelivr、Statically、gcore）

镜像 GitHub release 与 raw 内容的第三方 CDN。GitHub 自身被节流或地区封
锁时有用。

```sh
# jsDelivr（最常用）
curl -L https://cdn.jsdelivr.net/gh/x-cmd/x-cmd@main/README.md
curl -L https://cdn.jsdelivr.net/gh/x-cmd/x-cmd@v1.0.0/INSTALL.md

# Statically
curl -L https://cdn.statically.io/gh/x-cmd/x-cmd/main/README.md

# gcore
curl -L https://gcore.jsdelivr.net/gh/x-cmd/x-cmd@main/README.md
```

| 属性 | 值 |
| --- | --- |
| 速率限制 | **CDN 级别（无 per-user 限制；jsDelivr 免费层 ~5000 万 req/月）** |
| 覆盖 | 标签、分支、commit、semver 范围（`@1`、`@^1.2`） |
| 适用 | **GitHub 慢 / 被节流 / 不可用时的回退** |
| 代价 | 免费层有使用上限；付费层可用 |

**限制**：

- **最终一致性** —— 新 release 后 CDN 传播需要几秒到几分钟。用于
  时间敏感的时效性时不用。
- **无私有仓库** —— 仅公开仓库。
- **每文件大小限制** —— jsDelivr 上通常每文件 50 MB（与 GitHub 相似）。
- **CSP / SRI 摩擦** —— 如果你从 jsDelivr 加载，需要在 CSP 里添加
  `cdn.jsdelivr.net` 并使用 SRI hashes 保证安全。

### 暴露面对比 —— 四限矩阵

| 暴露面 | URL | 速率限制 | 软限制 | 适用 |
| --- | --- | --- | --- | --- |
| **Releases API** | `api.github.com/...` | 5000/小时 认证，60/小时 未认证 | 突发 → 二级 | 列出 releases |
| **HTML 抓取** | `github.com/.../releases/...` | 每 IP ~数百/小时 | DOM 漂移 | 一次性回退 |
| **Archive tarball** | `codeload.github.com/.../tar.gz/refs/...` | **CDN（Fastly）—— 无文档化的 GitHub 限制** | 仓库大小 ≤100 MB | 整个仓库快照 |
| **Raw 内容** | `raw.githubusercontent.com/...` | **CDN（Fastly）—— 无文档化的 GitHub 限制** | 每 IP ~60-100 req/分钟 | 单文件抓取 |
| **CDN 镜像** | `cdn.jsdelivr.net/gh/...` | CDN 级别（免费层 ~5000 万 req/月） | 最终一致 | 回退 / CDN |

### 机器人穿过的五种暴露面

对于想获取 `x-cmd/x-cmd` 每个 release 的工具：

```sh
# 第 1 步：Releases API —— 列出所有 releases（每页 1 个 API 调用）
curl -H "Authorization: Bearer $TOKEN" \
     'https://api.github.com/repos/x-cmd/x-cmd/releases?per_page=100&page=1'

# 第 2 步：对每个 release，用 archive 拉取 tarball（CDN，无 API 成本）
curl -L -o release.tar.gz \
     https://github.com/x-cmd/x-cmd/archive/refs/tags/v1.0.0.tar.gz

# 第 3 步：仓库内特定文件，用 raw（CDN，无 API 成本）
curl -L -o README.md \
     https://raw.githubusercontent.com/x-cmd/x-cmd/v1.0.0/README.md
```

API 预算：1 个列出请求。CDN 下载：无限。

**vs HTML 抓取**：每个 release 每个 ref 5 行。脆弱、未文档化限制、
无 auth、IP 分桶。

### 第四个角度：用 API 拉这三项时的速率限制

你可能问：*"如果我用 Releases API 列出，然后用 archive 拉 tarball，最后用 raw
拉单文件 —— 速率限制如何交互？"*

答案：**只有 Releases API 计数**。archive 与 raw 暴露面是 CDN 缓存，
不消耗 GitHub API 配额。所以：

| 调用 | 是否计入 GitHub 主限制？ |
| --- | --- |
| `api.github.com/...` | ✅ 是（5000/小时 认证 或 60/小时 未认证） |
| `github.com/.../archive/...`（tarball） | ❌ 否 |
| `raw.githubusercontent.com/...` | ❌ 否 |
| `cdn.jsdelivr.net/gh/...` | ❌ 否 |
| `github.com/.../releases/...`（HTML） | ⚠️ 是 —— 未文档化的 UI 限制 |

策略：**让 Releases API 做列出，让 `raw.xxx` 与 `codeload.github.com`
做下载**。

### 开发者角度 —— gitattributes + 云开发环境

如果你的目标是**项目本身作为代码**（不是 release 制品），并且你担心仓库大小、
submodules 或 LFS —— 不用 tarball。直接用 git，配 **gitattributes** 裁剪你
拉取的内容。

```sh
# Sparse checkout —— 只拉一个路径
git clone --depth=1 --filter=blob:none --sparse \
     https://github.com/x-cmd/x-cmd.git
cd x-cmd
git sparse-checkout set install.d docs

# 或者，用 .gitattributes 在你自己的仓库中裁剪 push 大小
# （让消费者拿到更小的 tarball）：
# .gitattributes
# install.d/*.tar.gz   filter=lfs diff=lfs merge=lfs -text
# large-dataset/*      filter=lfs diff=lfs merge=lfs -text
```

对于**云开发环境** —— 即"我甚至不想下载，我想要个沙箱"：

- **GitHub Codespaces** —— 官方；浏览器；适用 GH 配额。
- **Gitpod** —— 开源替代；适用任何 GH 仓库；免费层可用。
- **Google Colab** —— Python 优先；用 `!git clone https://github.com/x-cmd/x-cmd.git`
  克隆。不需要 release 下载。

这些环境**直接从 GitHub 流式拉取仓库**而不需要你取字节 —— 网络流量计入
GitHub 的 CDN，不计入你的 API 配额。

### 参考：`x eget` 故事

x-cmd [`eget`](https://github.com/x-bash/eget) 模块实现了这一策略：

1. 先用 **Releases API** 找到正确的制品 URL（1 个 API 调用）。
2. 如果 API 耗尽，**回退到 HTML 抓取** `github.com/<o>/<r>/releases/tag/<tag>`
   提取制品 URL（脆弱但速率限制分开）。
3. 通过 **`codeload.github.com` 或 `raw.githubusercontent.com`** 下载制品
   （CDN，无 API 配额）。

eget 流程记录在
[`x-bash/eget/lib/download`](https://github.com/x-bash/eget)。
关键不变量：**只有第 1 步动 API 预算**。

---

## 主 REST API 速率限制

大多数开发者撞到的"GitHub 速率限制"。

| 认证面 | 限制 | 单次 | 每 |
| --- | --- | --- | --- |
| 未认证 | 60 req | 1 小时 | 源 IP |
| 个人访问 token（PAT） | 5,000 req | 1 小时 | token |
| OAuth app（用户 token） | 5,000 req | 1 小时 | token |
| GitHub App user-to-server | 5,000+ req | 1 小时 | installation |
| GitHub App server-to-server | 5,000+ req | 1 小时 | installation |

5,000/小时上限在 OAuth / PAT / user-to-server 间相同。**GitHub Apps
可通过 `increasing-api-quota-for-github-apps` 申请表请求更高配额**，但
对典型工作流 5,000/小时够了。

### "Per token" 对 OAuth / PAT 的含义

配额是 **per token** 而非 per user。一个有 3 个 PAT 的用户拿到 3 个独立
的 5,000/小时 配额。这对 CI 隔离有用：每个工作流一个 PAT，每个工作流一个
预算。

### GitHub Apps：per-installation 配额

GitHub Apps 有 **per-installation** 配额 —— 5,000/小时计入 installation
而非 app 整体。如果你的 app 安装在 100 个 installation 跨 100 个仓库，
你实际上最多拿 500,000/小时 聚合，但每个 installation 的配额独立追踪。

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

上述查询扣 10 点（任一字段的最大值）。乘数按比例：connection 字段和聚合
字段比简单字段读取贵。

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

GitHub 在主限制之上强制 *secondary* 速率限制。它是启发式的 —— 由滥用样
式触发：

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
- **选对暴露面。** 列出 releases？用 API。下载？archive tarball。单文件？raw。

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
            # Secondary rate limit; sleep Retry-After exactly.
            time.sleep(retry_after)
            continue
        if response.status_code == 403:
            # X-RateLimit-Remaining == 0 means quota exhausted.
            if response.headers.get("X-RateLimit-Remaining") == "0":
                reset_at = int(response.headers["X-RateLimit-Reset"])
                wait = max(reset_at - time.time(), 1)
                time.sleep(min(wait, 3600))
                continue
        # Non-rate-limit error; let caller handle.
        response.raise_for_status()
    raise RateLimitExceeded()
```

三条经验：

1. **每次请求前预检 `X-RateLimit-Remaining`。** 如果是 0，别费心 —
   睡到 reset。
2. **存在 `Retry-After` 时尊重它。** Secondary 速率限制会设它；primary
   在 429 上也会设。
3. **选对暴露面。** 列出 releases？用 API。下载 tarball？用
   `codeload.github.com`。单文件？用 `raw.githubusercontent.com`。

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
- codeload.github.com（archive tarballs）：
  <https://docs.github.com/en/repositories/working-with-files/using-files/downloading-files-from-a-repository>
- raw.githubusercontent.com：
  <https://docs.github.com/en/repositories/working-with-files/using-files/viewing-and-understanding-files>
- jsDelivr GitHub 镜像：
  <https://www.jsdelivr.com/github/>
- x-bash/eget（上述策略的参考实现）：
  <https://github.com/x-bash/eget>

**验证状态**：截至 2024-11 验证以上内容。