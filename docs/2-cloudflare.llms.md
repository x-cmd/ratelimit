---
name: 2-cloudflare
description: Cloudflare has two distinct response signals that look similar but mean different things. **HTTP 429 Too Many Requests** is a rate limit (your code made too many API requests; honor `Retry-After`). **`cf-mitigated` is abuse detection** (Cloudflare thinks your client looks automated / bot-like; this is NOT a rate limit). Plus REST API quota (1200 req/5min, identical across plans) and per-product HTTP caps (Workers, KV, R2, D1) by plan tier.
type: reference
---

# Core Content

core_features:

  - REST API per-user quota: 1200 req / 5 min, **identical across Free/Pro/Business** plans
  - Per-product HTTP caps (zone / Workers / KV / R2 / D1) scale with plan tier; not the REST quota
  - Free tier: 100k req/day zone, 100k req/day Workers, 100k KV reads/day, 1k KV writes/day
  - **`HTTP 429` = rate limit.** Honor `Retry-After`, back off, retry.
  - **`cf-mitigated: challenge|block` = abuse detection.** NOT a rate limit. Requires different handling (slow down + change client identity).
  - Cloudflare's API does NOT consistently emit `RateLimit-*` headers — watch for 429 + Retry-After

# Key Information

highlights:

  - Per-user quota is keyed on **API token**, not user — two tokens = two independent quotas
  - Differentiation between plans is on consumer side (HTTP caps), not operator side (API quota)
  - Workers, KV, R2, D1 quotas are **separate** from REST API quota — filling one doesn't affect the other
  - **`cf-mitigated` is the abuse signal, distinct from `429` — needs different retry logic**
  - Retry-After is seconds (integer) when present
  - cf-mitigated triggers: User-Agent strings, automation cadence, IP reputation, headless signals

# Use Cases

use_cases:

  - Implementing a Cloudflare REST API client with proper retry / backoff
  - Deciding whether to upgrade plans based on which quota (REST or HTTP) is the bottleneck
  - Distinguishing rate-limit (429) from abuse-detection (cf-mitigated) responses in logs / alerts
  - Per-product capacity planning (Workers, KV, R2, D1)
  - Recovering from cf-mitigated by changing client identity (UA, IP, cadence)

# Related Resources

official:
  docs: https://developers.cloudflare.com/fundamentals/api/reference/limits/
  plans: https://www.cloudflare.com/plans
  cf_mitigated: https://developers.cloudflare.com/fundamentals/reference/protections/
related:
  bot_management: https://developers.cloudflare.com/bots/
  ddos: https://www.cloudflare.com/ddos/