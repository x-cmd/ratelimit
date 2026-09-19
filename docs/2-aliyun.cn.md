---
x-title: 阿里云 OpenAPI 速率限制 —— 默认 100 QPS 与 Throttling 错误码
x-desc: 阿里云 OpenAPI 默认每用户 100 QPS 的全局默认；不同产品（ECS/RAM/CDN）的特殊下限；非 RFC 6585 的错误码 `Throttling.User / Throttling.Api / Throttling.CloudBox`；客户端实现要点。
x-sidebar: 阿里云速率限制
x-keywords: aliyun, 阿里云, ratelimit, qps, openapi, throttling, error code, ecs, ram, cdn
x-json-ld:
  '@context': https://schema.org
  '@graph':
    - '@type': TechArticle
      headline: '阿里云 OpenAPI 速率限制'
      inLanguage: 'zh-CN'
      about: '阿里云各产品 API 速率限制与错误码'
---

# 阿里云 OpenAPI 速率限制

阿里云 API 网关的速率限制故事与 Cloudflare / GitHub 显著不同：
**没有标准的 RFC 6585 `Retry-After` 头**。阿里云用自定义错误码体
系区分不同层级的限流，对客户端实现有具体影响。

## 默认全局限额

| 项 | 值 | 范围 |
| --- | --- | --- |
| 默认 QPS | **100** | 每个用户每个 API |
| 时间窗 | 1 秒 | 滑动窗口 |

多数 OpenAPI 默认遵循此限。但**部分 API 设定了更低的具体下限**——
不能用"100 QPS 普适"假设所有接口。

## 各产品的具体限制（待 CI 核实）

| 产品 / API | 默认限 | 备注 |
| --- | --- | --- |
| ECS CreateInstance | 60 / 分钟 | 创建型操作通常更低 |
| ECS RunCommand | 较低 | 命令执行有额外管控 |
| RAM 用户 / 组 | 视产品页 | RAM 通常单独限额 |
| CDN 刷新 / 预热 | 100 / 天每域名 | 写操作每日限额 |

> ⚠️ 上表数字均标记 `verified: false`，需 CI scraper 与各
> 产品官方文档比对核实。**生产环境集成前请直接查产品页**。

## 错误码体系（与 RFC 6585 不同）

阿里云 OpenAPI 不返回标准 HTTP 429。当触发限流时，返回
`HTTP 400` 或 `HTTP 403`，响应体里通过 `Code` 字段标记具体
原因：

```json
{
  "RequestId": "...",
  "HostId": "...",
  "Code": "Throttling.User",
  "Message": "..."
}
```

`Code` 字段的三种取值的实际含义：

| Code | 触发场景 |
| --- | --- |
| `Throttling.User` | 当前用户级限流达到上限 |
| `Throttling.Api` | 当前 API 的全局限流达到上限（其他人也在打满） |
| `Throttling.CloudBox` | 实例 / Region 级别限流达到上限 |

**关键陷阱**：客户端不能只看 HTTP 状态码决定要不要重试 —— 必
须解析 `Code` 字段。

## 客户端实现要点

```python
import time
import json
import requests

def call_aliyun(action, params, ak, sk):
    for attempt in range(5):
        response = requests.post(
            f"https://{params.pop('product')}.aliyuncs.com",
            params={"Action": action, **params, ...}
        )
        body = response.json()
        code = body.get("Code", "")
        if not code.startswith("Throttling"):
            return body
        if code == "Throttling.User":
            time.sleep(1 + attempt)
        elif code == "Throttling.Api":
            time.sleep(5 + attempt * 2)
        elif code == "Throttling.CloudBox":
            time.sleep(60)
        else:
            raise RuntimeError(f"unexpected throttling: {code}")
    raise RuntimeError("rate limited after 5 tries")
```

三件事要注意：

1. **错误码分支处理。** `Throttling.User`（个人级）1 秒退避就够；
   `Throttling.Api`（API 级）可能需要更长；`Throttling.CloudBox`
   （资源级）可能需要几十秒。
2. **区分产品默认 vs 实际限额。** 100 QPS 是默认，但部分产品（ECS
   Create、RDS、SLB 等）有专门的低限额。要在 `data/<vendor>.yaml`
   里维护每个产品 / 接口的实际值。
3. **不要假定 Retry-After。** 阿里云不返回这个头；解析 `Code` 才是
   可靠路径。

## 与 Cloudflare / GitHub 的关键差异

| 项 | Cloudflare | GitHub | 阿里云 |
| --- | --- | --- | --- |
| HTTP 状态码 | 429 | 429 | 400 / 403 |
| 错误细节 | `Retry-After` 头 | `X-RateLimit-*` 头 | `Code` 字段 |
| 跨产品统一限额？ | 是（1200/5min） | 部分（核心 5000，搜索 30） | 否（每产品各自） |

## 与阿里云限流相关的产品页参考

- ECS API 限流：[help.aliyun.com/document_detail/25485.html](https://help.aliyun.com/document_detail/25485.html)
- 全局 OpenAPI 调用：见具体产品 API 参考首页的"使用限制"章节
- RAM 限流：见 RAM 产品文档的"API 调用限制"段落

具体数字需 CI scraper 抓取后写入 `data/aliyun.yaml`。