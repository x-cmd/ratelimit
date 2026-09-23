---
name: 4-aliyun
description: Aliyun OpenAPI rate limits — default 100 QPS per user per API, product-specific lower limits (ECS Create, RAM, CDN), non-RFC-6585 error codes (Throttling.User / Throttling.Api / Throttling.CloudBox) requiring body parsing rather than status-code handling.
type: reference
---

# Core Content

core_features:

- Default QPS: **100 per user per API**, 1-second sliding window
- Product-specific overrides: ECS CreateInstance 60/min, CDN refresh 100/day/domain
- **No standard RFC 6585 Retry-After header** — error code is in response body
- Three-tier throttling codes: `Throttling.User` (per-user), `Throttling.Api` (per-API), `Throttling.CloudBox` (per-resource)
- HTTP 400 or 403 (not 429) when rate-limited

## Key Information

highlights:

- Clients must parse the `Code` field in response body — HTTP status alone is insufficient
- Different Code values need different backoff strategies:
  - Throttling.User: 1s+ retry
  - Throttling.Api: 5s+ retry
  - Throttling.CloudBox: 60s+ retry (resource-level congestion)
- Account-level aggregate quotas exist; per-product rates stack against account limits
- No program-readable "current quota" API equivalent to Tencent's DescribeApiRateLimit

## Use Cases

use_cases:

- Implementing an Aliyun OpenAPI client that handles the three-tier throttling
- Distinguishing Throttling.User from Throttling.Api in observability (different team owns each tier)
- Pre-launch verification of per-product actual limits via official docs
- Migrating a client from "HTTP 429 + Retry-After" assumption to Aliyun's body-Code model

## Related Resources

official:
  docs: <https://help.aliyun.com/document_detail/146726.html>
  ecs: <https://help.aliyun.com/document_detail/25485.html>
related:
  errors: <https://help.aliyun.com/document_detail/315526.html>