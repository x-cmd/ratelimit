---
name: ratelimit
description: Vendor API rate limits and quota caps reference. Use when the user asks "what's the rate limit for X", "X QPS", "X API quota", "GitHub rate limit", "Cloudflare free tier limits", "how many requests per second can I make to Y", or wants to look up per-vendor HTTP 429 / RateLimit-* header conventions.
metadata: type=reference, source=team-curated, schema=yaml-per-vendor, refresh=manual, license=dual-(Apache-2.0+CC-BY-4.0), scope=api-rate-limits
---

# x-cmd/ratelimit — using the team's rate-limit reference

Two consumption paths. Pick whichever fits.

## 1. Direct YAML lookup (zero dependencies)

The structured data is the source of truth, in
`data/<vendor>.yaml`. Read it directly:

```sh
# Cloudflare per-user REST API quota
yq '.plans[].api_rate_limit' data/cloudflare.yaml

# GitHub REST authenticated quota
yq '.plans[] | select(.name == "Authenticated via PAT (REST)") | .api_rate_limit' data/github.yaml

# All vendors at once: how many of them have header-based rate-limit info?
yq '.header_conventions' data/*.yaml
```

`yq` is convenient but not required; YAML parses fine with
Python, Ruby, or `awk` for one-off greps:

```sh
# List every rate-limit surface covered
grep -rE 'surface:' data/ | sort -u | head -30
```

## 2. Per-vendor article (deep dive)

Each vendor has its own article under `docs/<n>-<vendor>.md`
(matching the `x-cmd/cve` article pattern):

- [`docs/2-github.en.md`](./docs/2-github.en.md) — REST + GraphQL +
  Actions + Search + secondary rate limits, header semantics.
- [`docs/3-cloudflare.en.md`](./docs/3-cloudflare.en.md) — REST API
  quota, per-product HTTP caps, `cf-mitigated` vs 429.
- [`docs/4-aliyun.en.md`](./docs/4-aliyun.en.md) — open API per-user
  QPS, the `Throttling.*` error code scheme.
- [`docs/5-tencent.en.md`](./docs/5-tencent.en.md) — Cloud API 3.0,
  `X-RateLimit-*` headers, `DescribeApiRateLimit`.
- [`docs/6-vercel.en.md`](./docs/6-vercel.en.md) — Function / Edge
  Function quotas, REST API 1 RPS default, RFC 9745 headers.
- [`docs/7-bandwagonhost.en.md`](./docs/7-bandwagonhost.en.md) — VPS
  port 25 block, bandwidth caps, connection limits; not an
  API-rate-limit story in the usual sense.
- [`docs/1-rate-limit-headers-cheatsheet.en.md`](./docs/1-rate-limit-headers-cheatsheet.en.md) —
  cross-vendor response-header matrix (`X-RateLimit-*`, `Retry-After`,
  `cf-mitigated`, IETF `RateLimit-*` draft).

## Common queries

```sh
# Cloudflare free tier's HTTP request cap
yq '.plans[0].product_limits[] | select(.surface | contains("HTTP"))' data/cloudflare.yaml

# GitHub headers — for client implementation
yq '.header_conventions' data/github.yaml

# Aliyun's non-standard error-code scheme
grep -A 5 "Throttling" docs/4-aliyun.en.md
```

## Schema (per `data/<vendor>.yaml`)

```yaml
vendor: <display name>
slug: <kebab-case>
api_base: <base URL or null>
docs_source: <URL of the official rate-limit doc>
verified: true | false          # whether the team has confirmed the numbers
last_verified: YYYY-MM-DD | null
pricing_notes: [...]
plans:
  - name: <plan display name>
    api_rate_limit:
      surface: <API or service name>
      limit: <number or "custom">
      window: <time window>
      per: <user|token|IP|installation|...>
      notes: <optional>
    product_limits:
      - surface: <product-level surface>
        limit: <number>
        window: <time window>
        notes: <optional>
    rate_limit_headers:
      ratelimit_remaining: <header name or null>
      notes: <optional>
header_conventions: ...        # cross-vendor header summary
```

## Verification state

Until the `.github/workflows/scrape.yml` job is built, every
YAML carries `verified: false` (except `github.yaml`, which
the team confirmed against official docs in 2024-11). The
numbers above the fold in the README are honest about that:

> ⚠️ Numbers above are subject to verification. Each
> data/<vendor>.yaml carries a verified flag and the
> official-docs URL; the CI scraper will refresh them
> against upstream on every release.

## License

- `LICENSE` (Apache 2.0): code, prose, scripts.
- `LICENSE-data` (CC-BY-4.0): data files under `data/`.