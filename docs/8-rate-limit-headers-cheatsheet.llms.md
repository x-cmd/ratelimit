---
name: 7-rate-limit-headers-cheatsheet
description: Cross-vendor HTTP rate-limit response header reference — Retry-After, X-RateLimit-*, IETF RateLimit-* draft (RFC 9745), per-vendor quirks (Aliyun has no headers, Tencent closest to draft, Vercel uses lowercase, GitHub uses X-prefix), recommended client patterns.
type: reference
---

# Core Content

core_features:

- `Retry-After` is universal RFC 6585 (seconds integer or HTTP-date)
- `X-RateLimit-Limit / Remaining / Reset` — historical style, X-prefixed, X-Reset is **epoch timestamp**
- `ratelimit-limit / remaining / reset` — Vercel lowercase (RFC 9745 style)
- `RateLimit: limit=100, remaining=50, reset=60` — IETF RFC 9745 single-header form
- `X-RateLimit-Resource` — GitHub-specific bucket identifier (core / search / graphql)
- `cf-mitigated` — Cloudflare-specific abuse signal, NOT a rate limit

## Key Information

highlights:

- Per-vendor: Cloudflare (Retry-After, cf-mitigated), Aliyun (no headers, Code body field), Tencent (X-RateLimit-* closest to draft), GitHub (X-RateLimit-* + Resource), Vercel (lowercase ratelimit-*), BandwagonHost (no API)
- IETF RFC 9745 finalizes the lowercase combined-header form
- `X-RateLimit-Reset` is epoch timestamp, NOT seconds-until-reset (most common client footgun)
- Three rules: always parse Retry-After if present, trust per-vendor signal before universal, track budget locally not via repeated requests
- Aliyun is the outlier — no standard rate-limit headers at all

## Use Cases

use_cases:

- Implementing a multi-vendor HTTP client with consistent rate-limit handling
- Auditing existing clients for proper handling of all 6 vendors' patterns
- Designing a rate-limit middleware that adapts per-vendor
- Migrating from X-RateLimit-* to RFC 9745 combined-header

## Related Resources

official:
  rfc_9745: <https://datatracker.ietf.org/doc/rfc9745/>
  rfc_6585: <https://datatracker.ietf.org/doc/html/rfc6585>
related:
  draft: <https://datatracker.ietf.org/doc/draft-ietf-httpapi-ratelimit-headers/>