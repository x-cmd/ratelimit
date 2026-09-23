---
name: 3-cloudflare
description: Cloudflare rate limits — read responses as (status code × cf-mitigated) matrix. 429 alone = REST API quota (1200 req/5min/token). 429 + cf-mitigated:rate-limit = zone-level rate rule. 403 + cf-mitigated:challenge|block|bot|ip|country = WAF/Bot/rule block. 200 + cf-mitigated:challenge = JS challenge page. Per-product HTTP caps (Workers/KV/R2/D1) are separate from REST API quota.
type: reference
---

# Core Content

core_features:

  - **status × cf-mitigated matrix** is the right way to read Cloudflare responses — not status code alone
  - 429 alone (no cf-mitigated) = REST API quota: 1200 req / 5 min / API token
  - 429 + cf-mitigated: rate-limit = zone-level rate-limit rule (not account-level quota)
  - 403 + cf-mitigated: challenge|block|bot|ip|country = WAF / Bot / rule block (NOT rate limit)
  - 200 + cf-mitigated: challenge = JS challenge page (HTML with embedded JS)
  - REST API quota is per-API-token, not per-account (multi-token = multi-quota)
  - Per-product HTTP caps (Workers, KV, R2, D1) are separate counters from REST API quota
  - Cloudflare does NOT emit X-RateLimit-* / X-RateLimit-Reset like GitHub does

# Key Information

highlights:

  - Two-axis reading: (HTTP status) × (cf-mitigated value) — not status alone
  - WAF/Bot blocks are NOT rate limits; they're Cloudflare protecting its website customers
  - Retry-After: integer seconds; only reliable signal on 429
  - Multi-token = independent quotas (CI isolation pattern)
  - Global API Key (legacy, pre-2024) is single-shared quota — CI teams migrate to API tokens to avoid collisions
  - cf-mitigated: block has no recovery path — must change client identity (UA, IP)
  - cf-mitigated: challenge usually solvable (CAPTCHA / JS)
  - Bot Management triggers cf-mitigated: bot (Enterprise / Super Bot Fight Mode)

# Use Cases

use_cases:

  - Reading Cloudflare API responses correctly via status × cf-mitigated
  - Distinguishing REST API quota exhaustion from zone rate-limit rules
  - Handling WAF/Bot blocks without auto-retry
  - Picking CI token isolation strategy (per-job tokens)
  - Capacity planning for Workers/KV/R2/D1 (separate from API quota)
  - Recovering from cf-mitigated blocks via UA/IP/cadence rotation
  - Solving JS challenge pages with headless browsers (puppeteer / playwright)

# Related Resources

official:
  api_limits: https://developers.cloudflare.com/fundamentals/api/reference/limits/
  workers_limits: https://developers.cloudflare.com/workers/platform/limits/
  kv_limits: https://developers.cloudflare.com/kv/platform/limits/
  r2_limits: https://developers.cloudflare.com/r2/platform/limits/
  d1_limits: https://developers.cloudflare.com/d1/platform/limits/
  protections: https://developers.cloudflare.com/fundamentals/reference/protections/
related:
  bot_management: https://developers.cloudflare.com/bots/
  ddos: https://www.cloudflare.com/ddos/