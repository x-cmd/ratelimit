---
name: 0-ratelimit-overview
description: x-cmd/ratelimit overview — vendor API rate-limit reference covering Cloudflare, Aliyun, Tencent Cloud, GitHub, Vercel, BandwagonHost. Data tables in data/<vendor>.yaml, long-form articles in docs/, dual-licensed (Apache 2.0 + CC-BY-4.0). **Project-agnostic; verified where flagged.**
type: summary
---

# Core Content

core_features:

- Reference table of API rate limits across 6 vendors: Cloudflare, Aliyun, Tencent Cloud, GitHub, Vercel, BandwagonHost
- Per-vendor YAML in `data/<vendor>.yaml` — source of truth, machine-readable
- Long-form articles in `docs/` (each in en + cn + llms + faq.yml)
- Dual license: Apache 2.0 (code, prose, scripts) + CC-BY-4.0 (data tables under `data/`)
- All YAMLs carry `verified: false` (except `github.yaml` verified 2024-11) until CI scraper refreshes upstream docs

## Key Information

highlights:

- README targets GitHub visitors; `docs/` are served at x-cmd.com/ratelimit
- Vendors covered: Cloudflare (REST + per-product), Aliyun (default 100 QPS + Throttling.* codes), Tencent (default 20 QPS + X-RateLimit-*), GitHub (5000/hr REST + 5000/hr GraphQL + 30/min Search + secondary), Vercel (GB-hours + Edge), BandwagonHost (port 25 + bandwidth)
- Cross-vendor reference: `docs/7-rate-limit-headers-cheatsheet.en.md` covers Retry-After, X-RateLimit-*, IETF RateLimit-* draft (RFC 9745)
- Aliyun is the outlier: no standard rate-limit headers, custom Code body field
- GitHub most-documented: 5000/hr REST, GraphQL points, secondary heuristic, X-RateLimit-* family

## Use Cases

use_cases:

- Looking up "Cloudflare per-user REST API quota" or "GitHub secondary rate limit triggers"
- Building a multi-vendor HTTP client that handles all rate-limit response shapes
- Auditing a project's existing rate-limit assumptions against current upstream docs
- Capacity planning (e.g., "if I'm on Vercel Hobby, how many invocations can I serve?")

## Related Resources

official:
  website: <https://x-cmd.com/ratelimit>
  repo: <https://github.com/x-cmd/ratelimit>
related:
  consumer: <https://github.com/x-cmd/x-cmd> (`mod/ratelimit/`) (forthcoming)
  standards: RFC 9745 (IETF RateLimit-* draft)
  inspiration: <https://github.com/x-cmd/cve>