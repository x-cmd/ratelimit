---
x-title: Vercel 速率限制 —— 函数执行、Edge 配额、REST API
x-desc: Vercel 各套餐的函数 / Edge 函数配额（Hobby 100 GB-hr/月，Pro 1 TB-hr/月）、REST API 默认 1 RPS、RFC 9745 小写 `ratelimit-*` 响应头，以及如何围绕函数执行上限做容量规划。
x-sidebar: Vercel 速率限制
x-keywords: vercel, ratelimit, gb-hr, serverless function, edge function, bandwidth, deployments, rfc 9745
x-json-ld:
  '@context': https://schema.org
  '@graph':
    - '@type': TechArticle
      headline: 'Vercel 速率限制'
      inLanguage: 'zh-CN'
      about: 'Vercel 函数、Edge 与 API 速率限制'
---

# Vercel 速率限制 —— 函数执行、Edge 配额、REST API

Vercel 的速率限制故事与 API-rate-limit-first 的厂商（Cloudflare
/GitHub）形状不同。主约束是**函数执行预算**（memory × time ×
调用次数）和**带宽**，API 速率限制反倒是次要的。

## 各套餐产品上限

| 上限 | Hobby（免费） | Pro | Enterprise |
| --- | --- | --- | --- |
| Serverless 函数执行 | 100 GB-hr / 月 | 1,000 GB-hr / 月 | 自定义 |
| Serverless 函数超时（默认） | 10s | 60s | 900s |
| Edge 函数调用 | 500,000 / 月 | 5,000,000 / 月 | 自定义 |
| Edge 函数代码大小 | 1 MB | 4 MB | 自定义 |
| 带宽（Fast Data Transfer） | 100 GB / 月 | 1 TB / 月 | 自定义 |
| 构建分钟 | 100 / 月 | 400 / 月 | 自定义 |
| 部署（生产） | 100 / 天 | 3,000 / 天 | 自定义 |
| 预览部署 | 不限 | 不限 | 不限 |

上表数字是训练数据期间各套餐已发布的限值；以
`vercel.com/docs/concepts/limits/overview` 实测为准。

## 理解 "GB-hours"

Vercel 的执行配额以 **GB-hours** 计算，定义为：

```
GB-hours = (内存，单位 GB) × (执行时间，单位小时)
         × (调用次数)
```

示例：1 GB 内存、100,000 次调用、平均每次 1 秒：

```
1 GB × (1 / 3600) 小时 × 100,000 次
= 27.78 GB-hours
```

Hobby 套餐 100 GB-hours 配额允许这个工作负载约 **3.6 倍**。Pro
套餐 36 倍。

## Vercel REST API

| 面 | 默认 | 每 |
| --- | --- | --- |
| Vercel REST API | 1 次 / 秒 | token |

多数 Vercel REST 端点共用这个默认。批量端点（`/v13/bulk-deployments`
等）通常允许更高。API 本身文档在 `vercel.com/docs/rest-api`。

## Vercel 响应头

Vercel 的 REST API 用小写 `RateLimit-*` 响应头，匹配 IETF draft
（RFC 9745）：

```http
ratelimit-limit: 60
ratelimit-remaining: 59
ratelimit-reset: 60
retry-after: 60
```

这是本仓库所有厂商里最干净的响应头格式 —— `retry-after` 是秒
数（整数），`ratelimit-*` 完全符合 IETF draft。

## 实用模式

### 选择内存与超时以匹配预算

如果你在 Hobby（100 GB-hr/月）且预期 1M 调用/月：

```
内存 × 每次执行时间 × 1M ≤ 100 GB-hours
```

1M 调用、100 GB-hr：

```
内存 × 时间 ≤ 3.6 × 10⁻⁴ GB-hour
         = 1.3 second-GB / 调用

所以 256 MB 函数 + 5s 超时 = 1.28 GB-second，
约 78k 调用/月就用完预算。
128 MB 函数 + 2s 超时 = 0.256 GB-second，
约 390k 调用/月还能跑。
```

权衡：低内存 = 单次成本低，但内存密集型路径 OOM 风险高。

### 区分"Hobby 用尽"与"代码错误"

Vercel 返回：

- 套餐超额（Hobby 100 GB-hr）→ HTTP 402
- 函数代码错误 → HTTP 500
- 函数超时 → HTTP 504

区分这些很关键：代码错误告警应该寻呼 on-call；套餐超额告
警应该通知账单 / 增长团队。

### Edge Functions vs Serverless Functions

**配额分开**。一个 Hobby 项目有：

- 100 GB-hr/月 给 Serverless Functions（Node.js / Python / Go /
  ...）
- 500,000 调用/月 给 Edge Functions（V8 isolate）

Edge Functions 单次更便宜，但有 1MB 代码大小硬限、不同的运行时
模型。Edge Functions 适合请求时中间件（鉴权、路由、A/B 测试），
Serverless Functions 适合算力密集型路径（图片处理、PDF 生成）。

## 本文不涵盖的

- Vercel 的 DDoS 防护（独立层）
- 构建镜像缓存上限
- 日志保留配额

都是不同话题；见 Vercel 文档。

## 来源

- 上限总览（函数 / Edge / 带宽 / 构建 / 部署）：
  <https://vercel.com/docs/concepts/limits/overview>
- REST API：
  <https://vercel.com/docs/rest-api>
- 价格：
  <https://vercel.com/pricing>
- Vercel Edge Functions 运行时：
  <https://vercel.com/docs/functions/edge-functions>
- Serverless Functions：
  <https://vercel.com/docs/functions/serverless-functions>
