---
name: 2-rate-limit-headers-cheatsheet
description: Cross-service rate-limit response headers quick reference. GitHub uses X-RateLimit-* family (Reset is UNIX epoch). Cloudflare uses Retry-After + cf-mitigated (NOT X-RateLimit-*). Search engines (Bing/Yahoo/DuckDuckGo/Baidu/Shenma) mostly bare 429. IETF RateLimit-* draft is the emerging standard.
type: reference


# Core Content

core_features:

  - "**Cross-service header matrix** in one table: GitHub / Cloudflare / Bing / IndexNow / Yahoo / DuckDuckGo / Baidu / Shenma"
  - "**GitHub X-RateLimit-* family** (Limit, Remaining, Reset, Used, Resource) — every response, not just 429"
  - "**GitHub X-RateLimit-Reset is UNIX epoch** — NOT seconds-until-reset (footgun)"
  - "**Cloudflare Retry-After + cf-mitigated** — two separate signals, not interchangeable"
  - "**Cloudflare does NOT send X-RateLimit-*** — don't look for it on CF"
  - "**Retry-After is the cross-service HTTP standard** — most services honor it"
  - "**Search engines mostly bare 429** — limited detail headers"
  - "**IETF RateLimit-* draft** — emerging standard (no X- prefix, Reset as relative seconds)"

# Key Information

highlights:

  - "**Two-axis reading for Cloudflare**: (status code) × (cf-mitigated value) — cf-mitigated is a header, can co-occur with 429 or 403"
  - "**GitHub 200 responses include rate-limit info** — pre-flight check before sending next request"
  - "**Cloudflare 403 + cf-mitigated: challenge|block|bot|ip|country is NOT a rate limit** — it's WAF / Bot protection"
  - "**Search engines (Bing/Yahoo/etc.)** — sparse detail, mostly just 429 status; conservative pacing needed"
  - "**cf-ray is the Cloudflare trace ID** — give to support for fast diagnosis"

# Use Cases

use_cases:

  - Reading rate-limit responses correctly across services
  - Distinguishing rate-limit (429) from WAF/Bot (403 + cf-mitigated) on Cloudflare
  - Pre-flight checking remaining quota on GitHub via response headers
  - Building cross-service retry logic that handles different header conventions
  - Diagnosing rate-limit issues with support (cf-ray for Cloudflare, X-RateLimit-* debug info for GitHub)
  - Migrating from custom rate-limit headers to IETF RateLimit-* standard

# Related Resources

official:
  github: https://docs.github.com/en/rest/using-the-rest-api/rate-limits-for-the-rest-api
  cloudflare_protections: https://developers.cloudflare.com/fundamentals/reference/protections/
  ietf_draft: https://datatracker.ietf.org/doc/draft-ietf-httpapis-ratelimit-headers/
related:
  github_article: docs/1-github.cn.md
  cloudflare_article: docs/3-cloudflare.cn.md