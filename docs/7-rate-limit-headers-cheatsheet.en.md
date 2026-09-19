---
x-title: Rate-limit headers cheatsheet — cross-vendor
x-desc: HTTP rate-limit response header conventions across vendors — Retry-After, X-RateLimit-*, IETF RateLimit-* draft (RFC 9745), and the differences in status-code (429 vs custom 400/403), window semantics, and reset semantics.
x-sidebar: Rate-limit headers cheatsheet
x-keywords: rate limit headers, ratelimit-limit, ratelimit-remaining, retry-after, x-ratelimit, rfc 9745, 429
x-json-ld:
  '@context': https://schema.org
  '@graph':
    - '@type': TechArticle
      headline: 'Rate-limit headers cheatsheet'
      inLanguage: 'en'
      about: 'HTTP rate-limit response header conventions across vendors'
---

# Rate-limit headers cheatsheet — cross-vendor

Different vendors implement rate-limit headers in different
shapes. This article is the cross-reference — what's
universal, what's custom, and what the IETF draft (RFC 9745)
says it should look like.

## Header cheat sheet

| Header | Meaning | IETF standard? |
| --- | --- | --- |
| `Retry-After` | Seconds until next request | RFC 6585 (universal) |
| `X-RateLimit-Limit` | Quota limit for the window | IETF draft uses `RateLimit-Limit` (no `X-` prefix) |
| `X-RateLimit-Remaining` | Requests remaining in window | IETF draft uses `RateLimit-Remaining` |
| `X-RateLimit-Reset` | **Epoch timestamp** (not seconds) | IETF draft uses `RateLimit-Reset` |
| `X-RateLimit-Resource` | Which bucket (`core`, `search`, etc.) | GitHub-specific |
| `ratelimit-limit` | Vercel lowercase | IETF draft compliant |
| `ratelimit-remaining` | Vercel lowercase | IETF draft compliant |
| `ratelimit-reset` | Vercel lowercase | IETF draft compliant |
| `RateLimit` | Combined: `limit=100, remaining=50, reset=60` | IETF draft |
| `cf-mitigated` | Cloudflare-specific abuse signal | not a rate limit |

## Vendor-by-vendor

| Vendor | Standard `Retry-After`? | Custom headers? | Trust semantics |
| --- | --- | --- | --- |
| Cloudflare | yes (most endpoints) | `cf-mitigated` | Rest of API quota is uniform across plans |
| 阿里云 | **no** (returns 400/403) | `Code` field in body | Three-tier throttling: User / Api / CloudBox |
| 腾讯云 | yes (integer seconds) | `X-RateLimit-*` | Closest to IETF draft of any vendor here |
| GitHub | yes (on 429) | `X-RateLimit-*`, `X-RateLimit-Resource` | Two-bucket: primary + secondary |
| Vercel | yes (lowercase) | `ratelimit-*` | RFC 9745-style headers |
| BandwagonHost | n/a | n/a | No programmatic API |

## The IETF draft (RFC 9745)

The IETF draft, finalized as RFC 9745 in 2024, defines a
single combined header:

```
HTTP/1.1 429 Too Many Requests
RateLimit: limit=100, remaining=50, reset=60
Retry-After: 60
```

- `RateLimit:` (single header) — three comma-separated
  key=value pairs: `limit`, `remaining`, `reset`.
- `Retry-After:` — seconds (integer) or HTTP-date.
- `reset` — seconds-until-reset (NOT epoch timestamp —
  this is the key fix vs `X-RateLimit-Reset`).

Vercel is closest to this format (lowercase `ratelimit-*`
headers per endpoint). Most other vendors still use the
historical `X-RateLimit-*` style with epoch timestamps.

## Three rules of thumb for client code

1. **Always parse `Retry-After` if present.** It's the
   most-portable signal: every vendor that emits a header
   at all emits this one. (Aliyun is the exception — they
   don't emit any standard rate-limit header at all.)

2. **Trust the per-vendor secondary signal before the
   universal one.** When GitHub returns 429 + `Retry-After:
   60` AND `X-RateLimit-Resource: search`, your search quota
   is what got exhausted, not your core REST quota.

3. **Track budget locally, not via repeated requests.**
   `X-RateLimit-Remaining` is authoritative but you don't
   need to make a request just to find out — store the
   counter locally and decrement on each request. Saves
   you from the bootstrap problem of "what's my quota at
   startup?".

## Why this matters

A robust HTTP client needs to handle all six vendors'
patterns — and Aliyun's "no header, custom Code field" case
is the easiest to forget until it bites you. The cheat
sheet above is the minimum surface area to test against in
any client library that claims "multi-vendor rate-limit
support".

## Sources

- IETF RFC 9745 (RateLimit-* header standard, finalized 2024):
  <https://datatracker.ietf.org/doc/rfc9745/>
- RFC 6585 (Retry-After origin):
  <https://datatracker.ietf.org/doc/html/rfc6585>
- IETF draft (historical):
  <https://datatracker.ietf.org/doc/draft-ietf-httpapi-ratelimit-headers/>
- Per-vendor sources for the cross-reference table: see articles
  1 (Cloudflare), 2 (Aliyun), 3 (Tencent), 4 (GitHub), 5 (Vercel),
  6 (BandwagonHost).

**Verification status**: standards-based, stable.
