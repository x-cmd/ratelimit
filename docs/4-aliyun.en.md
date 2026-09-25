---
x-title: Aliyun OpenAPI rate limits — 100 QPS default and the Throttling error codes
x-desc: Aliyun OpenAPI's default per-user 100 QPS rate limit; product-specific lower limits (ECS/RAM/CDN); the non-RFC-6585 error codes `Throttling.User / Throttling.Api / Throttling.CloudBox`; client implementation notes.
x-sidebar: Aliyun rate limits
x-keywords: aliyun, ratelimit, qps, openapi, throttling, error code, ecs, ram, cdn
x-json-ld:
  '@context': https://schema.org
  '@graph':
    - '@type': TechArticle
      headline: 'Aliyun OpenAPI rate limits'
      inLanguage: 'en'
      about: 'Aliyun OpenAPI per-product rate limits and error codes'
---

# Aliyun OpenAPI rate limits

Aliyun's rate-limit story differs from Cloudflare's and
GitHub's in one key way: **no standard RFC 6585 `Retry-After`
header**. Aliyun uses a custom error-code scheme to
distinguish which tier of rate limit was hit, and clients
need to parse the response body to handle each tier.

## Default global limit

| Field | Value | Scope |
| --- | --- | --- |
| Default QPS | **100** | per user, per API |
| Window | 1 second | sliding |

Most OpenAPI endpoints follow this default. But **some
APIs publish a lower specific limit** — you cannot assume
"100 QPS applies universally".

## Product-specific limits (pending CI verification)

| Product / API | Default limit | Notes |
| --- | --- | --- |
| ECS `CreateInstance` | 60 / minute | Create-class operations typically lower |
| ECS `RunCommand` | lower | Command execution has extra governance |
| RAM (users / groups) | varies | RAM typically has its own quota |
| CDN refresh / prefetch | 100 / day per domain | Write operations have daily caps |

> ⚠️ The numbers above are all marked `verified: false` —
> the CI scraper must pull each product's official doc and
> cross-check before publish. **Always verify against the
> live product page before launching integration.**

## Error-code scheme (not RFC 6585)

Aliyun OpenAPI does NOT return HTTP 429 for rate limiting.
On trigger, it returns `HTTP 400` or `HTTP 403`, with the
`Code` field in the response body specifying which tier
was hit:

```json
{
  "RequestId": "...",
  "HostId": "...",
  "Code": "Throttling.User",
  "Message": "..."
}
```

The three `Code` values:

| Code | Trigger |
| --- | --- |
| `Throttling.User` | Per-user rate limit reached |
| `Throttling.Api` | Per-API global rate limit reached (other tenants are also maxing) |
| `Throttling.CloudBox` | Per-instance / per-region rate limit reached |

**Key trap**: clients cannot decide whether to retry based
on HTTP status alone — the `Code` body field must be parsed.

## Client implementation notes

```python
import time
import json
import requests

def call_aliyun(action, params, ak, sk):
    for attempt in range(5):
        response = requests.post(
            f"https://{params.pop('product')}.aliyuncs.com",
            params={"Action": action, **params}
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

Three things to watch:

1. **Branch on the `Code` field.** `Throttling.User`
   (user-level) recovers in 1 second; `Throttling.Api`
   (API-level) needs longer; `Throttling.CloudBox`
   (resource-level) can need tens of seconds.
2. **Distinguish product default vs. actual limit.**
   100 QPS is the default but specific products (ECS
   Create, RDS, SLB) have dedicated lower limits. Maintain
   per-product actual values in `data/aliyun.yaml`.
3. **Don't rely on `Retry-After`.** Aliyun does not emit
   it; parsing the `Code` field is the only reliable path.

## Key differences vs Cloudflare / GitHub

| Field | Cloudflare | GitHub | Aliyun |
| --- | --- | --- | --- |
| HTTP status code | 429 | 429 | 400 / 403 |
| Error detail | `Retry-After` header | `X-RateLimit-*` headers | `Code` body field |
| Cross-product uniform quota? | yes (1200/5min) | partial (core 5000, search 30) | no (per product) |

## Reference links

- ECS API rate limits: <https://help.aliyun.com/document_detail/25485.html>
- General OpenAPI limits: see each product's "Usage limits" section
- RAM API limits: see RAM product docs

Specific numbers need CI scraper to write to `data/aliyun.yaml`.

## Sources

- General OpenAPI rate-limit doc:
  <https://help.aliyun.com/document_detail/146726.html>
- ECS API rate limits:
  <https://help.aliyun.com/document_detail/25485.html>
- RAM API limits: see RAM product doc index
  <https://help.aliyun.com/product/28625.html>
- CDN refresh / prefetch limits:
  <https://help.aliyun.com/document_detail/27256.html>
- Error code reference (`Throttling.*`):
  <https://help.aliyun.com/document_detail/315526.html>
