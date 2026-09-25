---
x-title: x-cmd/ratelimit —— 总览
x-desc: Cloudflare、阿里云、腾讯云、GitHub、Vercel、BandwagonHost 等厂商的 API 速率限制与配额上限速查。如何使用本仓库、YAML schema 是什么、各厂商文章怎么组织。**项目中立；标注已核实。**
x-sidebar: x-cmd/ratelimit 总览
x-keywords: ratelimit, 速率限制, api 配额, qps, rpm, x-cmd/ratelimit, 厂商 api, github 速率限制, cloudflare 速率限制
x-json-ld:
  '@context': https://schema.org
  '@graph':
    - '@type': TechArticle
      headline: 'x-cmd/ratelimit —— 厂商 API 速率限制'
      inLanguage: 'zh-CN'
      about: '厂商 API 速率限制速查'
---

# x-cmd/ratelimit —— 总览

团队日常使用最多的各厂商 API 速率限制与配额上限的速查。面
向已经知道 rate limiting 是什么、只是想查参数的工程师；想看
单厂商深度分析看 `docs/` 下对应文章，或直接读
`data/<vendor>.yaml` 拿机器可读版。

本页是一页摘要。下方文章深入 *数据结构怎么组织*、*每家厂
商的速率故事*、*客户端怎么写*（退避、响应头解析、fallback 策
略）。

## 仓库定位

- **工程师速查。** 你来查 "Cloudflare 每用户 API 配额" 或
  "GitHub 二级速率触发条件"。结构化数据在
  `data/<vendor>.yaml`；单厂商文章在 `docs/`。
- **可从上游刷新。** 每个 YAML 携带 `verified` 标记与
  `docs_source` 链接。未来 CI 任务会定时重抓官方文档、与
  YAML 比对、变更时开 PR。
- **双协议。** 源码、文章、脚本走 Apache 2.0（`LICENSE`）；
  `data/` 下的速率限制数据表走 CC-BY-4.0（`LICENSE-data`）—— 
  需要署名、商业可用。

## 已收录厂商

| 厂商 | 覆盖面 | 状态 | 文章 |
| --- | --- | --- | --- |
| **GitHub** | REST + GraphQL + Actions + 二级 + **五种下载策略（release / HTML / archive / raw / CDN）** | 已核实 2024-11 | [`2-github`](./2-github.en.md) |
| Cloudflare | REST API + 各产品 HTTP 配额 | 待核实 | [`3-cloudflare`](./3-cloudflare.en.md) |
| 阿里云 | OpenAPI 各产品 QPS | 待核实 | [`4-aliyun`](./4-aliyun.en.md) |
| 腾讯云 | Cloud API 3.0 | 待核实 | [`5-tencent`](./5-tencent.en.md) |
| Vercel | 函数/Edge + REST API | 待核实 | [`6-vercel`](./6-vercel.en.md) |
| BandwagonHost | VPS 端口 / 带宽 / 连接上限 | 待核实 | [`7-bandwagonhost`](./7-bandwagonhost.en.md) |
| 跨厂商 | HTTP 速率响应头、backoff | n/a | [`1-rate-limit-headers-cheatsheet`](./1-rate-limit-headers-cheatsheet.en.md) |

## 怎么使用

### 直接查 YAML

数据是权威源，YAML 格式。

```sh
# Cloudflare 每用户 REST API 配额
yq '.plans[].api_rate_limit' data/cloudflare.yaml

# GitHub REST 认证后配额
yq '.plans[] | select(.name == "Authenticated via PAT (REST)") | .api_rate_limit' data/github.yaml

# 所有厂商响应头一览
yq '.header_conventions' data/*.yaml
```

`yq` 用着方便但非必需；YAML 用 Python / Ruby / `awk` 解析都行。

### 单厂商深度文章

每个厂商有一篇专文，讲实际细节（响应头语义、错误码约定、坑）：

- [`2-github`](./2-github.en.md) —— REST + GraphQL + Actions + Search + 二级速率、响应头语义
- [`3-cloudflare`](./3-cloudflare.en.md) —— REST 配额、`cf-mitigated` 与 429 的区别
- [`4-aliyun`](./4-aliyun.en.md) —— 开放 API 每用户 QPS、`Throttling.*` 错误码体系
- [`5-tencent`](./5-tencent.en.md) —— Cloud API 3.0、`X-RateLimit-*` 响应头、`DescribeApiRateLimit`
- [`6-vercel`](./6-vercel.en.md) —— Function / Edge Function 配额、REST API 默认 1 RPS、RFC 9745 头
- [`7-bandwagonhost`](./7-bandwagonhost.en.md) —— VPS 端口 25 屏蔽、带宽上限、连接上限
- [`1-rate-limit-headers-cheatsheet`](./1-rate-limit-headers-cheatsheet.en.md) —— 跨厂商 HTTP 速率响应头速查

## 核实状态

CI scraper 任务还没建起来前，每个 YAML 都带 `verified` 标记。
`github.yaml` 已核实（2024-11）；其他都 `verified: false`，等
CI scraper 第一次跑完后由团队手动切换。

## 进一步阅读

- [`RATELIMIT-RESEARCH.md`](../../RATELIMIT-RESEARCH.md) ——
  工作笔记，含核实状态。
- [`SKILL.md`](../../SKILL.md) —— AI agent 用法。
- [`CONTRIBUTING.md`](../../CONTRIBUTING.md) —— 贡献流程。

## 来源

本文引用的数据来源详见文章 1–7 各厂商段。跨厂商标准：

- IETF RFC 9745（RateLimit-* 头标准，2024 定稿）：
  <https://datatracker.ietf.org/doc/rfc9745/>
- RFC 6585（Retry-After 出处）：
  <https://datatracker.ietf.org/doc/html/rfc6585>
