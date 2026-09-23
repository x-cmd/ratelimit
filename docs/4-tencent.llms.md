---
name: 3-tencent
description: Tencent Cloud API 3.0 rate limits — 20 QPS default per user per API (tighter than Aliyun's 100), X-RateLimit-* headers (closest to IETF RateLimit-* draft among Chinese cloud providers), DescribeApiRateLimit for programmatic quota lookup.
type: reference
---

# Core Content

core_features:

- Default QPS: **20 per user per API**, 1-second sliding window
- Account-level aggregate: ~1000 QPS across all APIs combined
- `X-RateLimit-Limit / Remaining / Window` headers (closest match to IETF RateLimit-* draft)
- `Retry-After` header (integer seconds, on 429 only)
- `DescribeApiRateLimit` API for programmatic quota lookup

## Key Information

highlights:

- 20 QPS tighter than Aliyun's 100 — default assumption should be 20, not optimistic
- Headers: `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Window`, `Retry-After` (on 429)
- Window is 1 second — single failed request + 1s sleep usually enough; exponential backoff unnecessary
- `DescribeApiRateLimit` returns per-API config: `{ApiName, MaxRequestNum, Strategy, WindowSeconds}`
- Major products: CVM 50-100 QPS, CDB 50 QPS, COS higher, CDN varies

## Use Cases

use_cases:

- Pre-launch verification of actual quota via DescribeApiRateLimit
- Implementing a Tencent Cloud API client with X-RateLimit-* parsing
- Monitoring account-level and API-level aggregate quota
- Designing client libraries with standard IETF-style header parsing

## Related Resources

official:
  docs: <https://cloud.tencent.com/document/product/301/30495>
  describe_api_rate_limit: <https://cloud.tencent.com/document/api/306/7234>
related:
  pricing: <https://cloud.tencent.com/pricing>