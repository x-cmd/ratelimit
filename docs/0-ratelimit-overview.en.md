---
x-title: x-cmd/ratelimit — overview
x-desc: A reference of API rate limits and quota caps across Cloudflare, Aliyun, Tencent Cloud, GitHub, Vercel, and BandwagonHost. How to consume the data, how to read the YAML schema, and how the per-vendor articles are organized. **Project-agnostic; verified where flagged.**
x-sidebar: x-cmd/ratelimit overview
x-keywords: ratelimit, rate limit, api quota, qps, rpm, x-cmd/ratelimit, vendor api, github rate limit, cloudflare rate limit
x-json-ld:
  '@context': https://schema.org
  '@graph':
    - '@type': TechArticle
      headline: 'x-cmd/ratelimit — vendor API rate limits'
      inLanguage: 'en'
      about: 'Vendor API rate limit reference'
---

# x-cmd/ratelimit — overview

A reference for API rate limits and quota caps across the
services the team uses most. Designed for engineers who
already know what rate limiting is and just want the
parameters — see the per-vendor articles for deep dives,
or jump straight to `data/<vendor>.yaml` for the machine-
readable version.

This page is the one-page summary. The articles below go
deeper on *how* the data is structured, *what* each
vendor's rate-limit story looks like, and *how* to handle
rate limits in client code (backoff, header parsing,
fallback strategies).

## What this repo is

- **Quick reference for engineers.** You came here to look
  up "Cloudflare's per-user API quota" or "GitHub's
  secondary rate-limit triggers". The structured data is
  under `data/<vendor>.yaml`; per-vendor articles are under
  `docs/`.
- **Refreshable from upstream.** Each YAML carries a
  `verified` flag and a `docs_source` URL. A future
  workflow job will re-pull official docs on a schedule,
  diff against the YAML, and open a PR when upstream
  numbers move.
- **Two-license.** Code, prose, scripts under Apache 2.0
  (`LICENSE`). The data tables under `data/` under
  CC-BY-4.0 (`LICENSE-data`) — attribution required,
  commercial use OK.

## Vendors covered

| Vendor | Surface | Status | Article |
| --- | --- | --- | --- |
| **GitHub** | REST + GraphQL + Actions + secondary + **5 download strategies (release / HTML / archive / raw / CDN)** | verified 2024-11 | [`1-github`](./1-github.en.md) |
| Cloudflare | REST API + per-product HTTP caps | unverified | [`3-cloudflare`](./3-cloudflare.en.md) |
| 阿里云 (Aliyun) | OpenAPI per-product QPS | unverified | [`4-aliyun`](./4-aliyun.en.md) |
| 腾讯云 (Tencent Cloud) | Cloud API 3.0 | unverified | [`5-tencent`](./5-tencent.en.md) |
| Vercel | Function / Edge + REST API | unverified | [`6-vercel`](./6-vercel.en.md) |
| BandwagonHost | VPS-level caps | unverified | [`7-bandwagonhost`](./7-bandwagonhost.en.md) |
| Cross-vendor | HTTP rate-limit headers, backoff | n/a | [`2-rate-limit-headers-cheatsheet`](./2-rate-limit-headers-cheatsheet.en.md) |

## How to consume

### Direct YAML lookup

The data is the source of truth, in YAML.

```sh
# Cloudflare per-user REST API quota
yq '.plans[].api_rate_limit' data/cloudflare.yaml

# GitHub REST authenticated quota
yq '.plans[] | select(.name == "Authenticated via PAT (REST)") | .api_rate_limit' data/github.yaml

# All vendors' header conventions at a glance
yq '.header_conventions' data/*.yaml
```

`yq` is convenient but not required; YAML parses fine with
Python, Ruby, or `awk`.

### Per-vendor article deep dive

Each vendor has its own article covering practical patterns
specific to that vendor (header semantics, error code
conventions, gotchas):

- [`1-cloudflare`](./1-cloudflare.md) — REST quota,
  `cf-mitigated` vs 429 distinction.
- [`2-aliyun`](./2-aliyun.md) — open API per-user QPS, the
  `Throttling.*` error code scheme.
- [`3-tencent`](./3-tencent.md) — Cloud API 3.0,
  `X-RateLimit-*` headers, `DescribeApiRateLimit`.
- [`4-github`](./4-github.md) — REST + GraphQL + Actions
  + Search + secondary rate limits, header semantics.
- [`6-vercel`](./6-vercel.en.md) — Function / Edge Function
  quotas, REST API 1 RPS default, RFC 9745 headers.
- [`7-bandwagonhost`](./7-bandwagonhost.en.md) — VPS port 25
  block, bandwidth caps, connection limits.
- [`7-rate-limit-headers-cheatsheet`](./7-rate-limit-headers-cheatsheet.md) —
  HTTP-rate-limit header conventions across vendors, with
  cross-reference back to per-vendor articles.

## Verification status

Until the CI scraper is built, every YAML carries a
`verified` flag. `github.yaml` is verified (2024-11); the
rest are `verified: false` and pending the first scraper
run. The team manually cross-checks before flipping the
flag.

## Where to read next

- [`RATELIMIT-RESEARCH.md`](../../RATELIMIT-RESEARCH.md) —
  working notes, including what the team has verified vs.
  what's still pending.
- [`SKILL.md`](../../SKILL.md) — AI-agent recipes.
- [`CONTRIBUTING.md`](../../CONTRIBUTING.md) — how to
  contribute a new vendor or correct an existing entry.

## Sources

This article references the per-vendor sources listed in each
of articles 1–7. Cross-vendor standards:

- IETF RFC 9745 (RateLimit-* header draft, finalized 2024):
  <https://datatracker.ietf.org/doc/rfc9745/>
- RFC 6585 (Retry-After origin):
  <https://datatracker.ietf.org/doc/html/rfc6585>
