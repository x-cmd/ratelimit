---
x-title: Tencent Cloud API 3.0 rate limits — 20 QPS default and X-RateLimit-* headers
x-desc: Tencent Cloud API 3.0's per-user 20 QPS default; the `X-RateLimit-Limit / Remaining / Window` headers (closest to the IETF `RateLimit-*` draft among Chinese cloud providers); and `DescribeApiRateLimit` for programmatic quota lookup.
x-sidebar: Tencent Cloud rate limits
x-keywords: tencent, ratelimit, qps, cloud api 3.0, x-ratelimit-limit, x-ratelimit-remaining, describleapiratelimit
x-json-ld:
  '@context': https://schema.org
  '@graph':
    - '@type': TechArticle
      headline: 'Tencent Cloud API 3.0 rate limits'
      inLanguage: 'en'
      about: 'Tencent Cloud API 3.0 rate limits and response headers'
---

# Tencent Cloud API 3.0 rate limits

Tencent Cloud's Cloud API 3.0 follows the "default 20 QPS
+ standard `X-RateLimit-*` headers" approach — among
Chinese cloud providers, this is **the closest match to the
IETF `RateLimit-*` draft**. Cleaner client code than
Aliyun, more standardized than Cloudflare.

## Default global limit

| Field | Value | Scope |
| --- | --- | --- |
| Default QPS | **20** | per user, per API |
| Window | 1 second | sliding |
| Account-level aggregate | 1,000 QPS | all APIs combined (typical) |

20 QPS is tighter than Aliyun's 100 QPS. **Default
assumptions should be 20**, not optimistic values like
100 or 200.

## Product-specific limits (pending CI verification)

| Product | Default limit | Notes |
| --- | --- | --- |
| CVM (Cloud Virtual Machine) | typically 50–100 QPS | varies by instance / region |
| CDB (Cloud Database) | typically 50 QPS | write operations lower |
| COS (Object Storage) | higher | independent SDK path |
| CDN | varies by endpoint | refresh / prefetch have daily caps |
| VPC | typically 50 QPS | |

Specific numbers need `data/tencent.yaml` to be paired
with `DescribeApiRateLimit` for verification.

## Response headers (closest to IETF `RateLimit-*` draft)

Tencent Cloud API on most endpoints returns:

```http
X-RateLimit-Limit: 20
X-RateLimit-Remaining: 18
X-RateLimit-Window: 1
Retry-After: 1
```

- `X-RateLimit-Limit` — current window ceiling (20)
- `X-RateLimit-Remaining` — remaining calls in window
- `X-RateLimit-Window` — window length in seconds
- `Retry-After` — backoff seconds on 429 only

The naming is almost identical to RFC 9745's
`RateLimit-Limit / Remaining / Reset`, just with the
`X-` prefix.

## `DescribeApiRateLimit`: programmatic quota lookup

Tencent Cloud exposes `DescribeApiRateLimit` to **read
your account's actual quota configuration** for each API.
This is useful for:

- Pre-launch: query every API you'll touch, confirm the
  actual limits (not the default assumption of 20)
- Monitoring: periodic runs of `DescribeApiRateLimit` to
  track account-level and API-level quotas

```sh
tccli cam DescribeApiRateLimit \
  --ApiName "DescribeInstances"
```

Sample response:

```json
{
  "ApiName": "DescribeInstances",
  "MaxRequestNum": 20,
  "Strategy": "RegionLevel",
  "WindowSeconds": 1
}
```

## Client implementation notes

```python
import time
import requests

def call_tencent(url, headers, max_retries=5):
    for attempt in range(max_retries):
        response = requests.get(url, headers=headers)
        if response.status_code != 429:
            return response
        retry_after = int(response.headers.get("Retry-After", "1"))
        # Tencent Cloud's Retry-After is integer seconds
        time.sleep(retry_after)
    raise RateLimitExceeded()
```

Three notes:

1. **`Retry-After` is integer seconds**, not HTTP-date
   strings.
2. **Window is 1 second.** A single failed request + 1
   second sleep is usually enough. Exponential backoff is
   unnecessary for Tencent; simple sleep + retry works.
3. **Use `X-RateLimit-Remaining` for budget tracking.**
   Don't actively retry to probe quota — a failed request
   burns one quota unit.

## Key differences vs Aliyun / Cloudflare / GitHub

| Field | Cloudflare | GitHub | Aliyun | Tencent |
| --- | --- | --- | --- | --- |
| Default QPS | 1200/5min | 5000/hr | 100/sec | **20/sec** |
| HTTP status | 429 | 429 | 400/403 | 429 |
| Error detail | `Retry-After` | `X-RateLimit-*` | `Code` field | `X-RateLimit-*` |
| Header format | custom | full draft | none | **closest to draft** |

## Reference links

- General API rate limit docs: <https://cloud.tencent.com/document/product/301/30495>
- `DescribeApiRateLimit`: <https://cloud.tencent.com/document/api/306/7234>

Specific numbers need CI scraper to write to
`data/tencent.yaml`.

## Sources

- General Cloud API 3.0 rate-limit doc:
  <https://cloud.tencent.com/document/product/301/30495>
- `DescribeApiRateLimit` API reference:
  <https://cloud.tencent.com/document/api/306/7234>
- Per-product rate limits (look in each product's API doc
  under "API Call Limits" or "Usage Limits" — Tencent Cloud docs
  typically label these as 调用限制 / 使用限制):
  - CVM: <https://cloud.tencent.com/document/product/213>
  - CDB: <https://cloud.tencent.com/document/product/236>
  - COS: <https://cloud.tencent.com/document/product/436>
  - VPC: <https://cloud.tencent.com/document/product/215>
