---
x-title: 腾讯云 API 3.0 速率限制 —— 20 QPS 默认与 X-RateLimit-* 头
x-desc: 腾讯云 Cloud API 3.0 默认每用户 20 QPS；`X-RateLimit-Limit / Remaining / Window` 头（最接近 IETF RateLimit-* draft 的中国云厂）；`DescribeApiRateLimit` 接口用于程序化读取自己的限额。
x-sidebar: 腾讯云速率限制
x-keywords: tencent, 腾讯云, ratelimit, qps, cloud api 3.0, x-ratelimit-limit, x-ratelimit-remaining, describleapiratelimit
x-json-ld:
  '@context': https://schema.org
  '@graph':
    - '@type': TechArticle
      headline: '腾讯云 Cloud API 3.0 速率限制'
      inLanguage: 'zh-CN'
      about: '腾讯云各产品 API 速率限制与响应头'
---

# 腾讯云 API 3.0 速率限制

腾讯云 Cloud API 3.0 走"默认 20 QPS + 通用 `X-RateLimit-*` 头"的
路线 —— 在中国云厂里属于**最贴近 IETF `RateLimit-*` draft 的实
现**。客户端写起来比阿里云省事，比 Cloudflare 标准化。

## 默认全局限额

| 项 | 值 | 范围 |
| --- | --- | --- |
| 默认 QPS | **20** | 每用户每 API |
| 时间窗 | 1 秒 | 滑动窗口 |
| 账户级聚合上限 | 1,000 QPS | 所有 API 合计（典型值） |

20 QPS 比阿里云的 100 QPS 紧。**默认假设要写成"20"**，不要
拿 100 / 200 这种乐观值。

## 各产品的具体限制（待 CI 核实）

| 产品 | 默认限 | 备注 |
| --- | --- | --- |
| CVM（云服务器） | 通常 50–100 QPS | 视实例 / 区域 |
| CDB（云数据库） | 通常 50 QPS | 写操作更低 |
| COS（对象存储） | 较高 | 走独立 SDK |
| CDN | 视接口 | 刷新 / 预热有日上限 |
| VPC | 通常 50 QPS | |

具体数字需 `data/tencent.yaml` 与 `DescribeApiRateLimit` 联动核
实。

## 响应头（最像 IETF `RateLimit-*` draft）

腾讯云 API 在大多数 endpoint 返回：

```http
X-RateLimit-Limit: 20
X-RateLimit-Remaining: 18
X-RateLimit-Window: 1
Retry-After: 1
```

- `X-RateLimit-Limit` —— 当前窗口上限（20）
- `X-RateLimit-Remaining` —— 剩余可调用次数
- `X-RateLimit-Window` —— 窗口长度（秒）
- `Retry-After` —— 限流触发时的退避秒数（429 才有）

跟 RFC 9745 的 `RateLimit-Limit / Remaining / Reset` 形态几乎
一致，只是字段名带 `X-` 前缀。

## `DescribeApiRateLimit`：程序化读取自己的限额

腾讯云提供 `DescribeApiRateLimit` API，**返回当前账号每个 API
当前的限流配置**。这条对以下场景特别有用：

- 上线前用 `DescribeApiRateLimit` 查一遍你关心的 API，确认实
  际限额（不是默认假设的 20）
- 监控自己的限额 —— 周期跑这个 API，看账户级 / API 级聚合
  是多少

```sh
tccli cam DescribeApiRateLimit \
  --ApiName "DescribeInstances"
```

返回示例：

```json
{
  "ApiName": "DescribeInstances",
  "MaxRequestNum": 20,
  "Strategy": "RegionLevel",
  "WindowSeconds": 1
}
```

## 客户端实现要点

```python
import time
import requests

def call_tencent(url, headers, max_retries=5):
    for attempt in range(max_retries):
        response = requests.get(url, headers=headers)
        if response.status_code != 429:
            return response
        retry_after = int(response.headers.get("Retry-After", "1"))
        # 腾讯云 Retry-After 是秒数（整数）
        time.sleep(retry_after)
    raise RateLimitExceeded()
```

注意：

1. **`Retry-After` 是整数秒**，不是 HTTP-date 字符串。
2. **窗口是 1 秒**。失败 1 秒后立即重试通常就过了。指数退避
   对腾讯云不必要，简单 sleep + 重试即可。
3. **优先用 `X-RateLimit-Remaining` 做"配额查询"**。不要主动
   重试来探测 quota —— 一次失败就消耗一次配额。

## 与阿里云 / Cloudflare / GitHub 的关键差异

| 项 | Cloudflare | GitHub | 阿里云 | 腾讯云 |
| --- | --- | --- | --- | --- |
| 默认 QPS | 1200/5min | 5000/hr | 100/sec | **20/sec** |
| HTTP 状态码 | 429 | 429 | 400/403 | 429 |
| 错误细节 | `Retry-After` | `X-RateLimit-*` | `Code` 字段 | `X-RateLimit-*` |
| 头部格式 | 自定义 | 完整 draft | 无 | **最像 draft** |

## 与腾讯云限流相关的产品页参考

- 通用 API 限流说明：[cloud.tencent.com/document/product/301/30495](https://cloud.tencent.com/document/product/301/30495)
- `DescribeApiRateLimit`：[cloud.tencent.com/document/api/306/7234](https://cloud.tencent.com/document/api/306/7234)

具体数字需 CI scraper 抓取后写入 `data/tencent.yaml`。

## 来源

- 通用 Cloud API 3.0 速率限制文档：
  <https://cloud.tencent.com/document/product/301/30495>
- `DescribeApiRateLimit` API 参考：
  <https://cloud.tencent.com/document/api/306/7234>
- 各产品速率限制（在各产品 API 文档的"调用限制"或"使用限制"段）：
  - CVM：<https://cloud.tencent.com/document/product/213>
  - CDB：<https://cloud.tencent.com/document/product/236>
  - COS：<https://cloud.tencent.com/document/product/436>
  - VPC：<https://cloud.tencent.com/document/product/215>
