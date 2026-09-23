---
name: 1-cloudflare
description: Cloudflare rate limits — REST API per-user quota (1200 req/5min across plans), per-product HTTP caps (Workers, KV, R2, D1) varying by plan tier, the distinction between 429 (rate limit) and cf-mitigated (abuse defense), and retry strategy.
type: reference
---

# Core Content

core_features:

- REST API per-user quota: 1200 req / 5 min, **identical across Free/Pro/Business** plans
- Per-product HTTP caps (zone / Workers / KV / R2 / D1) scale with plan tier; not the REST quota
- Free tier: 100k req/day zone, 100k req/day Workers, 100k KV reads/day, 1k KV writes/day
- `cf-mitigated: challenge|block` is **NOT** a rate limit — it's abuse defense, requires different handling
- Cloudflare's API does NOT consistently emit `RateLimit-*` headers — watch for 429 + Retry-After
- Per-endpoint exceptions (e.g., GET /zones/:id, DNS read) have higher quotas

## Key Information

highlights:

- Per-user quota is keyed on **API token**, not user — two tokens = two independent quotas
- Differentiation between plans is on consumer side (HTTP caps), not operator side (API quota)
- Workers, KV, R2, D1 quotas are **separate** from REST API quota — filling one doesn't affect the other
- `cf-mitigated` is the abuse signal, distinct from 429 — needs different retry logic
- Retry-After is seconds (integer) when present

## Use Cases

use_cases:

- Implementing a Cloudflare REST API client with proper retry / backoff
- Deciding whether to upgrade plans based on which quota (REST or HTTP) is the bottleneck
- Distinguishing rate-limit (429) from abuse-detection (cf-mitigated) responses in logs / alerts
- Per-product capacity planning (Workers, KV, R2, D1)

## Related Resources

official:
  docs: <https://developers.cloudflare.com/fundamentals/api/reference/limits/>
  plans: <https://www.cloudflare.com/plans>
related:
  bot_management: <https://developers.cloudflare.com/bots/>
  ddos: <https://www.cloudflare.com/ddos/>