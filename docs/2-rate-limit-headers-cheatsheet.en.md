---
x-title: Rate-limit response headers — cross-service quick reference
x-desc: Cross-service quick reference: GitHub (X-RateLimit-*), Cloudflare (Retry-After + cf-mitigated), Bing/IndexNow, Yahoo, DuckDuckGo, Baidu, Shenma — what headers each service uses to signal rate limiting.
x-sidebar: Rate-limit response headers
x-keywords: ratelimit, response headers, retry-after, x-ratelimit, cf-mitigated, http headers, 429, 403
x-json-ld:
  '@context': https://schema.org
  '@graph':
    - '@type': TechArticle
      headline: 'Rate-limit response headers'
      inLanguage: 'en'
      about: 'Cross-service rate-limit response header comparison'
---

# Rate-limit response headers — cross-service quick reference

Read rate-limit responses via **HTTP response headers**. Different services use different header families, but the patterns are similar.

---

## Quick reference

### What headers each service uses to signal rate limiting

| Service | Status | Header | Meaning |
| --- | --- | --- | --- |
| **GitHub** | 200 / 403 | `X-RateLimit-Limit` / `Remaining` / `Reset` / `Used` / `Resource` | Current bucket state (every response) |
| **GitHub** | 429 | `Retry-After` | Integer seconds to wait |
| **Cloudflare** | 429 | `Retry-After` | Integer seconds |
| **Cloudflare** | 200 / 403 | `cf-mitigated: challenge\|block\|rate-limit\|bot\|ip\|country` | WAF / Bot detection signal |
| **Cloudflare** | any | `cf-ray` | Internal trace ID; give to CF support |
| **Bing / IndexNow** | 429 | bare 429, possibly `Retry-After` | — |
| **Yahoo** | 429 | bare 429 | — |
| **DuckDuckGo** | 429 | bare 429 | — |
| **Baidu** | 429 | bare 429 | — |
| **Shenma** | 429 | bare 429 | — |

**Key patterns**:

- **`Retry-After` is the cross-service convention** — most services follow this HTTP standard
- **`X-RateLimit-*` is GitHub's family** — other services don't use it
- **`cf-mitigated` is Cloudflare-specific** — only appears at CF edge
- **Bing / Yahoo / DuckDuckGo / Baidu / Shenma**: mostly bare 429 status, sparse detail headers

### Header family comparison

| Family | Used by | Fields |
| --- | --- | --- |
| `X-RateLimit-*` | GitHub | `Limit` / `Remaining` / `Reset` / `Used` / `Resource` |
| `Retry-After` (HTTP standard) | GitHub / Cloudflare / most | Integer seconds |
| `cf-mitigated` | Cloudflare | `challenge` / `block` / `rate-limit` / `bot` / `ip` / `country` |
| `cf-ray` | Cloudflare | Trace ID |
| `RateLimit-*` (IETF draft) | Few newer services | `limit` / `remaining` / `reset` / `policy` |

**Trend**: industry moving toward IETF `RateLimit-*` draft, but GitHub's `X-RateLimit-*` and Cloudflare's `cf-mitigated` are still custom implementations.

---

## Writeup

### GitHub: `X-RateLimit-*` family

Sent on every response (not just 429):

```
HTTP/1.1 200 OK
X-RateLimit-Limit: 5000
X-RateLimit-Remaining: 4999
X-RateLimit-Reset: 1640000000
X-RateLimit-Used: 1
X-RateLimit-Resource: core
```

- **`X-RateLimit-Reset` is UNIX epoch** (not "seconds until reset"). Compute: `wait = max(reset - now, 1)`
- **`X-RateLimit-Resource`** tells you which bucket (`core` / `search` / `graphql` / `integration_manifest` / etc.)
- **`Retry-After`** only on 429 — relative seconds

Full details in [1-github](1-github) §3.

### Cloudflare: `Retry-After` + `cf-mitigated`

Cloudflare uses two separate signals:

```
HTTP/1.1 429 Too Many Requests
Retry-After: 60
```

```
HTTP/1.1 403 Forbidden
cf-mitigated: challenge
cf-ray: 6c8a...
```

- **`429` + `Retry-After`**: true rate limit (quota reached)
- **`cf-mitigated: challenge|block|bot|ip|country`**: WAF / Bot triggered — **NOT rate limit**
- **`cf-ray`**: give to Cloudflare support for fast diagnosis

**Important**: Cloudflare does **NOT** send `X-RateLimit-*` — don't go looking for it.

Full details in [3-cloudflare](3-cloudflare) §1.

### Bing / IndexNow / Yahoo / DuckDuckGo / Baidu / Shenma

Search engines (other than Google) are mostly bare:

- **429** status (occasionally 503)
- Maybe `Retry-After`, maybe not
- **No** detailed `X-RateLimit-*` family

Practical:
- On 429, sleep (30 sec to a few min)
- Conservative concurrency (≤ 1 QPS per IP)
- Use proxy / IP pool to spread

Specific numbers in each service's dedicated article: [3-cloudflare](3-cloudflare) / future Bing / Yahoo / DuckDuckGo / Baidu / Shenma.

### IETF `RateLimit-*` draft

Industry moving toward IETF's `RateLimit-*` family:

```
RateLimit-Limit: 100
RateLimit-Remaining: 50
RateLimit-Reset: 30
```

- Similar to GitHub but **no `X-` prefix**
- `Reset` is **relative seconds** (unlike GitHub's absolute epoch)
- Few newer services use it; mainstream services still on custom

---

## How to read a rate-limit response (5 steps in practice)

Regardless of service, when you see 429 / 403:

1. **Check status code** — 429 vs 403 vs 200-with-cf-mitigated
2. **Check `Retry-After`** — present means "wait this long"
3. **Check `X-RateLimit-*`** (if service uses it) — compute remaining time and reset
4. **Check `cf-mitigated`** (if behind Cloudflare) — distinguish rate limit from WAF
5. **Check `cf-ray` or service-specific trace ID** — give to support for fast diagnosis

---


---

## Sources

- GitHub rate-limit headers: <https://docs.github.com/en/rest/using-the-rest-api/rate-limits-for-the-rest-api>
- Cloudflare protections / headers: <https://developers.cloudflare.com/fundamentals/reference/protections/>
- IETF RateLimit headers draft: <https://datatracker.ietf.org/doc/draft-ietf-httpapis-ratelimit-headers/>