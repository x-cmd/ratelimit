---
x-title: Cloudflare rate limits — REST API and per-product caps
x-desc: Cloudflare's per-user REST API rate limit, free-tier product caps (HTTP requests, Workers), the difference between API rate limits and the cf-mitigated challenge header, and how to implement a robust client retry strategy.
x-sidebar: Cloudflare rate limits
x-keywords: cloudflare, ratelimit, qps, api quota, workers, free tier, cf-mitigated, retry-after, 429
x-json-ld:
  '@context': https://schema.org
  '@graph':
    - '@type': TechArticle
      headline: 'Cloudflare rate limits'
      inLanguage: 'en'
      about: 'Cloudflare API and per-product rate limits'
---

# Cloudflare rate limits — REST API and per-product caps

Cloudflare has two distinct rate-limit stories that
sometimes get conflated:

1. **REST API quota** — per-user request budget against
   `api.cloudflare.com`. Affects anyone calling the
   Cloudflare API.
2. **Per-product HTTP caps** — total HTTP requests served
   by a zone, Workers invocations, KV operations, etc.
   Affects customers of those products.

Plus a third thing that *looks* like rate limiting but isn't:
the **`cf-mitigated` challenge header**. Documented
separately at the end of this article.

## REST API quota

Most Cloudflare REST API endpoints share a single per-user
quota:

| Plan | Limit | Window | Per |
| --- | --- | --- | --- |
| Free | 1200 req | 5 min | user |
| Pro | 1200 req | 5 min | user |
| Business | 1200 req | 5 min | user |
| Enterprise | custom | custom | user |

The per-user quota is the same across plans; what changes
is the product-level HTTP caps (next section). The
`1200 req / 5 min` for "Free" and "Pro" is intentionally
identical — Cloudflare's pricing differentiation is on
the *consumer side* of the API, not the operator side.

### What "per user" means here

Cloudflare's API rate limit is keyed on the **API token**
(or API key, for legacy integrations). A user with two API
tokens gets two independent quotas — the tokens don't
share. This is intentional: CI jobs can each get their own
token to isolate bursts.

### Endpoints with separate quotas

A small number of endpoints have their own quotas instead
of the shared 1200/5min:

- `GET /zones/:id` (zone details) — higher per-endpoint
  quota to support zone-listing workflows.
- DNS read endpoints — historically higher quotas to
  support bulk enumeration.
- Workers KV / R2 / D1 — these are *per-product* quotas
  (covered below), not REST API quotas.

Verify the per-endpoint exceptions against the live
`developers.cloudflare.com/fundamentals/api/reference/limits/`
page; new exceptions are added occasionally.

## Per-product HTTP caps (free tier)

| Product | Free | Pro | Business | Enterprise |
| --- | --- | --- | --- | --- |
| HTTP requests / zone / day | 100,000 | 10M / month | 100M / month | custom |
| Workers requests / day | 100,000 | 1M / month | 20M / month | custom |
| Pages requests | unlimited (bandwidth cap) | unlimited (bandwidth cap) | unlimited (bandwidth cap) | custom |
| KV reads / day | 100,000 | 10M | 100M | custom |
| KV writes / day | 1,000 | 1M | 10M | custom |
| KV deletes / day | 1,000 | 1M | 10M | custom |
| R2 operations / month | 10M (A-class) | 50M | custom | custom |
| D1 reads / day | 5M | 5B (rows) | custom | custom |

These caps are **separate** from the REST API quota —
filling your Workers quota doesn't affect your API quota
and vice versa. Watch both.

## `cf-mitigated` vs 429

Cloudflare has two distinct response signals:

- **`429 Too Many Requests`** — your code made too many
  API requests. Honor `Retry-After` and back off.
- **`cf-mitigated: challenge` or `cf-mitigated: block`**
  — Cloudflare detected abusive patterns from your client
  (User-Agent strings, automation patterns, request
  frequency, etc.) and served a challenge or block page.
  This is **not** a rate limit; it's an abuse defense.

The `cf-mitigated` case is harder to recover from
automatically — the client typically needs to slow down
fundamentally (real human-like pacing) or use a different
network. The 429 case is just "slow down a bit".

### Practical advice

```python
# Pseudo-code: distinguish the two
response = requests.get(...)
if response.status_code == 429:
    sleep(int(response.headers.get("Retry-After", "60")))
    retry()
elif response.headers.get("cf-mitigated"):
    # An actual abuse mitigation. Don't retry immediately;
    # back off significantly or rotate identity.
    log.warning(f"cf-mitigated: {response.headers['cf-mitigated']}")
    raise AbusiveRequestDetected(...)
```

## Client-side retry strategy

A robust Cloudflare client:

```python
import time
import random

def call_cloudflare(url, token, max_retries=5):
    for attempt in range(max_retries):
        response = requests.get(url, headers={"Authorization": f"Bearer {token}"})
        if response.status_code != 429:
            return response
        # Exponential backoff with jitter
        backoff = (2 ** attempt) + random.uniform(0, 1)
        time.sleep(min(backoff, 60))  # cap at 60s
    raise RateLimitExceeded()
```

Three rules of thumb:

1. **Honor `Retry-After`** if present — Cloudflare sets it
   on 429s; respecting it avoids hitting the limit again
   immediately.
2. **Exponential backoff with jitter** — bare exponential
   backoff causes thundering-herd retries when many
   clients hit the limit simultaneously.
3. **Per-token budget tracking** — Cloudflare gives you
   1200/5min, but your token might be shared across
   processes. Track usage locally to avoid 429s.

## What this article does NOT cover

- DDoS protection (separate from rate limiting).
- Bot management (Cloudflare's Bot Fight Mode / Super
  Bot Fight Mode / Bot Management for Enterprise).
- Rate limiting rules you set yourself on your own zone
  (these are configurations on your domain, not Cloudflare
  account-wide limits).

These are different layers; see Cloudflare's docs for
each.

## Sources

- REST API per-user limits:
  <https://developers.cloudflare.com/fundamentals/api/reference/limits/>
- Per-product HTTP caps:
  - Workers: <https://developers.cloudflare.com/workers/platform/limits/>
  - KV: <https://developers.cloudflare.com/kv/platform/limits/>
  - R2: <https://developers.cloudflare.com/r2/platform/limits/>
  - D1: <https://developers.cloudflare.com/d1/platform/limits/>
- Free / Pro / Business / Enterprise pricing:
  <https://www.cloudflare.com/plans>
- `cf-mitigated` (WAF / abuse-detection semantics):
  <https://developers.cloudflare.com/fundamentals/reference/protections/>
