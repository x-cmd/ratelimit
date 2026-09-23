---
x-title: Cloudflare rate limits — quick reference (status code + cf-mitigated)
x-desc: Quick reference: REST API quota (1200 req/5min/token), per-product HTTP caps (Workers/KV/R2/D1), status code + cf-mitigated header matrix. How to tell from a response whether Cloudflare is rate-limiting or WAF/Bot-blocking, and how to handle each.
x-sidebar: Cloudflare rate limits
x-keywords: cloudflare, ratelimit, qps, api quota, workers, free tier, cf-mitigated, retry-after, 429, 403
x-json-ld:
  '@context': https://schema.org
  '@graph':
    - '@type': TechArticle
      headline: 'Cloudflare rate limits'
      inLanguage: 'en'
      about: 'Cloudflare API and per-product rate limits'
---

# Cloudflare rate limits — quick reference (status code + cf-mitigated)

---

## Quick reference

### 1. status code × cf-mitigated — how to read a response

| HTTP status | `cf-mitigated` | Meaning | Fix |
| --- | --- | --- | --- |
| `429` | (none) | **REST API quota** (1200/5min/token) | Slow down + `Retry-After` |
| `429` | `rate-limit` | **Zone-level rate-limit rule** | Slow down + `Retry-After` |
| `403` | `challenge` | **WAF challenge** | Solve / change pattern |
| `403` | `block` | **WAF direct block** | Change cadence + UA / IP |
| `403` | `bot` | **Bot Management triggered** | Same as `block` |
| `403` | `ip` | **IP-based rule** | Rotate IP |
| `403` | `country` | **Country-based rule** | Rotate IP |
| `200` | `challenge` | **JS challenge page** (HTML) | Run JS in headless browser |

`challenge` / `block` / `bot` / `ip` / `country` are all WAF (Web Application Firewall) / Bot Management actions — Cloudflare protecting **the websites it serves**, **not** limiting your API quota.

### 2. REST API quota (source of 429 with no cf-mitigated)

| Plan | Limit | Window | Unit |
| --- | --- | --- | --- |
| Free | 1200 req | 5 min | API token |
| Pro | 1200 req | 5 min | API token |
| Business | 1200 req | 5 min | API token |
| Enterprise | Custom | Custom | API token |

**Unit is API token, not Cloudflare account.**

- One token = one independent quota. Two tokens = two quota pools.
- Give each CI job its own token — a runaway job doesn't take out the fleet.
- Global API Key (legacy, pre-2024) is one shared key — legacy CI hits 429 because of this.

Source: [developers.cloudflare.com/fundamentals/api/reference/limits/](https://developers.cloudflare.com/fundamentals/api/reference/limits/)

### 3. Per-product HTTP caps (separate from REST API quota)

| Product | Free | Pro | Business |
| --- | --- | --- | --- |
| HTTP requests / zone / day | 100K | 10M / mo | 100M / mo |
| Workers requests / day | 100K | 1M / mo | 20M / mo |
| Pages requests | Unlimited (bandwidth cap) | Unlimited (bandwidth cap) | Unlimited (bandwidth cap) |
| KV reads / day | 100K | 10M | 100M |
| KV writes / day | 1K | 1M | 10M |
| KV deletes / day | 1K | 1M | 10M |
| R2 operations / month | 10M (Class A) | 50M | Custom |
| D1 reads / day | 5M | 5B rows / mo | Custom |

Workers / KV / R2 / D1 are per-product counters — **separate** from REST API quota. Burning Workers doesn't touch API quota. Enterprise is all-custom — talk to sales.

Sources: [Workers](https://developers.cloudflare.com/workers/platform/limits/) · [KV](https://developers.cloudflare.com/kv/platform/limits/) · [R2](https://developers.cloudflare.com/r2/platform/limits/) · [D1](https://developers.cloudflare.com/d1/platform/limits/)

### 4. Key response headers

| Header | Meaning | When |
| --- | --- | --- |
| `Retry-After` | Integer seconds; wait this long before retrying | On 429 |
| `cf-mitigated` | See matrix above | Various WAF / quota scenarios |
| `cf-ray` | CF internal trace ID; give this when contacting CF support | On any error |
| `cf-cache-status` | Cache hit / miss | On any response |

Cloudflare does **NOT** emit `X-RateLimit-*` / `X-RateLimit-Reset` like GitHub does. **429 + `Retry-After` is the only reliable rate-limit signal**.

---

## Writeup

### 1. `429` — rate limit, but check cf-mitigated for the source

#### `429` + (no cf-mitigated)

Most common. This is the **REST API quota** reaching its limit: 1200 req / 5 min / API token. Standard rate-limit handling:

1. **Honor `Retry-After`** (integer seconds) — don't retry before that time.
2. **Exponential backoff with jitter** — `base * 2^attempt + random(0, jitter)`. Bare exponential causes thundering herd.
3. **Track quota locally** — know you're approaching the cap before hitting 429.
4. **Multi-token isolation** — each CI job gets its own token; a runaway doesn't take out the fleet.

`429` is **recoverable** — slow down, window rolls over, continue.

#### `429` + `cf-mitigated: rate-limit`

This is a **zone-level rate-limit rule** triggering — not the account-level API quota. Rule examples:

- 60 requests / minute per IP
- 10 requests / 10 seconds per path
- 5 requests / 5 minutes per country

Same fix (slow down + `Retry-After`), but different **cause** — your domain's rule, not Cloudflare's global quota. See the rule config in Cloudflare dashboard → Security → Rate limit rules.

### 2. `403` + `cf-mitigated` — not rate limit, it's WAF / Bot Management

Every value of `cf-mitigated` in a 403 (`challenge` / `block` / `bot` / `ip` / `country`) is **not** a rate limit — it's Cloudflare's WAF / Bot Management triggered.

#### Why Cloudflare blocks you

Cloudflare isn't picking on you. **Cloudflare is protecting its customers (websites using Cloudflare) from suspicious traffic.**

- Website customers pay precisely so their sites serve "real humans only".
- WAF / Bot Management protects that value.
- You (the calling client) get blocked because your client pattern triggered the protection layer Cloudflare set up for website customers.

Once you understand this, "slow down + change UA + change IP" isn't a workaround — it's what Cloudflare designed the system to encourage.

#### Triggers (heuristics; CF doesn't publish exact thresholds)

- **User-Agent** — `python-requests/2.31.0`, `curl/8.4.0`, etc. are naked signals. Real browsers send `Mozilla/5.0 ...`.
- **Cadence** — perfect 1-second intervals, no human-like jitter.
- **Per-IP QPS** — sustained high QPS against the same endpoint.
- **Headless browser fingerprints** — missing plugins, missing fonts, anomalous canvas / WebGL.
- **IP reputation** — datacenter IPs, Tor exits, residential proxies with bad history.

#### `challenge` vs `block`

- `challenge` — Cloudflare gives you a CAPTCHA or JS challenge. **Solving usually lets you continue.**
- `block` — Cloudflare blocks directly, no recovery path. **You must change client pattern or IP.**

#### `bot` value

`bot` is Bot Management product (Enterprise or Super Bot Fight Mode) triggering. Not triggered unless explicitly enabled.

### 3. `200` + `cf-mitigated: challenge` — JS challenge page

Cloudflare returns `200 OK` with HTML that looks normal but contains an embedded JS challenge. The client must execute the JS to compute a cookie (`cf_clearance`) that lets subsequent requests through.

- Headless browsers (puppeteer / playwright) can run it — built-in JS engine.
- `curl` / `requests` can't by default — no JS engine. Use `cloudscraper` / `undetected-chromedriver`, or give up.

### 4. Client handling

```python
import time
import random

def call_cloudflare(url, token, max_retries=5):
    for attempt in range(max_retries):
        r = requests.get(url, headers={"Authorization": f"Bearer {token}"})

        if r.status_code == 429:
            # Rate limit — honor Retry-After, back off
            wait = int(r.headers.get("Retry-After", "60"))
            time.sleep(wait + random.uniform(0, 5))
            continue

        if "cf-mitigated" in r.headers:
            # WAF / Bot — don't auto-retry; change pattern or IP
            raise AbuseDetected(r.headers["cf-mitigated"])

        return r

    raise RateLimitExceeded()
```

Points:

1. **`429`: strictly honor `Retry-After`** — don't retry before that time.
2. **Any `cf-mitigated`: don't auto-retry** — WAF is pattern-matching; same pattern = same block.
3. **`429`: exponential backoff with jitter** — to avoid thundering herd.
4. **Multi-token CI isolation** — each token is an independent quota pool.
5. **Monitor `cf-mitigated` ratio** — a spike means client pattern or IP reputation changed, not a quota problem.

---

## Not covered here

- **DDoS protection** — separate layer from rate limiting.
- **Bot Management products** (Bot Fight Mode / Super Bot Fight Mode / Enterprise Bot Management) — CF-sold add-ons, off by default.
- **Your zone-level rate-limit rule config** — see Cloudflare dashboard.
- **WAF IP allow/block lists** — different layer.

See Cloudflare's own docs for each.