# x-cmd/ratelimit —— 各厂商 API 速率限制速查

团队日常使用最多的各厂商 API 速率限制与配额上限的速查。
本 README 是给落到 GitHub 仓库的访客看的；长文与各厂商深
度文章放在 `docs/`（被作为 `x-cmd.com/ratelimit` 的官方
站点发布）。

## 仓库定位

- **给工程师的速查。** 你来查 "Cloudflare 每用户 API 配额"
  或 "GitHub 的二级速率限制触发条件"。结构化数据在
  `data/<vendor>.yaml`；厂商级文章在 `docs/`。
- **可从上游刷新。** 每个 YAML 携带 `verified` 标记与
  `docs_source` 链接。未来的 `.github/workflows/scrape.yml`
  任务会定时重抓官方文档、与 YAML 比对、变更时开 PR。
- **双协议。** 源码、文章、脚本走 Apache 2.0
  （`LICENSE`）。`data/` 下的速率限制数据表走
  CC-BY-4.0（`LICENSE-data`）—— 需要署名、商业可用。

## 已收录厂商

| 厂商 | 覆盖面 | 状态 | 文章 |
| --- | --- | --- | --- |
| GitHub | REST + GraphQL + Actions + 二级限制 | 已核实 2024-11 | [`docs/2-github.en.md`](./docs/2-github.en.md) |
| Cloudflare | REST API + 各产品 HTTP 配额 | 待核实 | [`docs/3-cloudflare.en.md`](./docs/3-cloudflare.en.md) |
| 阿里云 | OpenAPI 各产品 QPS | 待核实 | [`docs/4-aliyun.en.md`](./docs/4-aliyun.en.md) |
| 腾讯云 | Cloud API 3.0 速率限制 | 待核实 | [`docs/5-tencent.en.md`](./docs/5-tencent.en.md) |
| Vercel | 函数/Edge 配额 + REST API | 待核实 | [`docs/6-vercel.en.md`](./docs/6-vercel.en.md) |
| BandwagonHost | VPS 端口 / 带宽 / 连接上限 | 待核实 | [`docs/7-bandwagonhost.en.md`](./docs/7-bandwagonhost.en.md) |
| 跨厂商 | HTTP 速率响应头、backoff | n/a | [`docs/1-rate-limit-headers-cheatsheet.en.md`](./docs/1-rate-limit-headers-cheatsheet.en.md) |

## 一览表

| 厂商 | 主要 API 限额 | 时间窗 | 响应头 |
| --- | --- | --- | --- |
| Cloudflare（REST） | 1200 次 | 5 分钟 / 用户 | `Retry-After`、`cf-mitigated` |
| 阿里云（开放 API） | 100 QPS | 1 秒 / 用户 | 自定义 `Code` 字段，非 RFC 6585 |
| 腾讯云（API 3.0） | 20 QPS | 1 秒 / 用户 | `X-RateLimit-*`、`Retry-After` |
| GitHub REST（PAT） | 5000 次 | 1 小时 / token | `X-RateLimit-*`、`Retry-After` |
| GitHub REST（未认证） | 60 次 | 1 小时 / IP | `X-RateLimit-*`、`Retry-After` |
| GitHub GraphQL | 5000 点 | 1 小时 / token | `X-RateLimit-*`、`Retry-After` |
| GitHub Search | 30 次 | 1 分钟 / 用户 | `X-RateLimit-*` |
| Vercel REST | 1 次 | 1 秒 / token | 小写 `ratelimit-*`（RFC 9745 风格） |

> ⚠️ 上表数字均待核实。每个 `data/<vendor>.yaml` 携带
> `verified` 标记与官方文档链接；CI 抓取脚本会在每次发布时
> 与上游重新比对。

## 进一步阅读

- **`data/<vendor>.yaml`** —— 机器可读，真源
- **`docs/0-ratelimit-overview.en.md`** —— "如何使用本仓库"权威参考
- **`docs/<n>-<vendor>.md`** —— 单厂商深度文章
- **`RATELIMIT-RESEARCH.md`** —— 工作笔记（包含核实状态）

## 协议

- `LICENSE` —— Apache 2.0（代码、文章、脚本）
- `LICENSE-data` —— CC-BY-4.0（`data/` 下的数据）

## 贡献 —— 欢迎参与

欢迎外部 PR 修改数据、补充厂商、改文章；团队会对每次改动签字。**建议先 issue、再 PR** —— 先开 issue 让 maintainer 看一下范围，确认后再发 PR。

三个 issue 模板，挑最贴近的一个：

| 想做什么 | Issue 类型 |
| --- | --- |
| 新增一个厂商 / API 表面进来监控 | [`monitor-target`](https://github.com/x-cmd/ratelimit/issues/new?template=monitor-target.yml) |
| 修一个错的 / 过期的速率数字 | [`errata`](https://github.com/x-cmd/ratelimit/issues/new?template=errata.yml) |
| 改文档 / FAQ / schema / 其它任何建议 | [`other`](https://github.com/x-cmd/ratelimit/issues/new?template=other-suggestion.yml) |

> **AI agent 创建 issue** —— 模板里以 `type: <三类之一>` 开头，每类都有专门的提示。**先读提示块**；任何声明必须给官方文档 URL；**在 maintainer 确认范围之前不要开 PR**。

完整流程见 [`CONTRIBUTING.md`](./CONTRIBUTING.md)。