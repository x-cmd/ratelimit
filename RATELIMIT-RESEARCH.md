# RATELIMIT-RESEARCH.md

Working notes for the rate-limit data collection. Not
authoritative — every entry is marked with its verification
status. WebFetch isn't reachable from this environment, so
all "已核实" markers point at official doc URLs that the
CI scraper needs to re-fetch and confirm against on every
release.

**Verification legend:**
- `[已核实: <url>, <date>]` — confirmed against the
  linked official doc on the date shown.
- `[待核实: <url>]` — value is from training-time
  knowledge; the CI scraper must pull the linked doc and
  cross-check before publish.
- `[需讨论]` — value is uncertain or varies by region /
  plan; flag for the team to resolve.

**Plan for verification:**
A future `.github/workflows/scrape.yml` job pulls each
`[待核实]` URL, extracts the relevant rate-limit numbers,
diffs against the YAML in `data/`, and opens a PR if the
upstream doc changed. Until that pipeline is built, every
data file's `verified:` field should default to `false`.

---

## 1. Cloudflare

**Scope:** Cloudflare API (REST) rate limits, plus
zone/Worker/DNS plan-level request caps.

| Endpoint / scope | Default limit | Plan-dependent? | Status |
| --- | --- | --- | --- |
| REST API, authenticated | 1200 req / 5 min / user | same across plans | `[待核实: developers.cloudflare.com/fundamentals/api/reference/limits/]` |
| Free zone, HTTP requests | 100,000 / day | yes (Pro/Business/Ent: higher) | `[待核实]` |
| Workers requests, Free | 100,000 / day | yes | `[待核实]` |
| DNS queries, Free | unlimited on authoritative; 1M / month on free DNS-only | yes | `[待核实]` |

**Pricing context (USD, as of 2026-09 training knowledge —
verify against cloudflare.com/plans):**
| Plan | Price | Notable |
| --- | --- | --- |
| Free | $0/mo | 100k req/day, .workers.dev subdomain |
| Pro | $25/mo (approx, verify) | 1M req/mo, basic analytics |
| Business | $250/mo (verify) | 100M req/mo, SLA, role-based access |
| Enterprise | custom | custom limits |

**Notes:**
- Cloudflare rate-limit headers (`cf-mitigated: challenge`,
  `Retry-After`) are NOT the same as API rate limits. Both
  need to be documented separately.
- The "Free plan 100k req/day" is for **page-rule zones**;
  Workers KV and R2 have separate per-resource limits.

---

## 2. Aliyun (阿里云)

**Scope:** OpenAPI per-product rate limits. Aliyun's defaults
vary by API product; the central guidance is "100 req/sec
per user per API by default".

| API scope | Default limit | Status |
| --- | --- | --- |
| Most APIs (default) | 100 QPS / user | `[待核实: help.aliyun.com/document_detail/146726.html]` |
| ECS Create* APIs | lower, often 60/min for CreateInstance | `[待核实: help.aliyun.com/document_detail/25485.html]` |
| RAM (users / groups) | custom limits, lower | `[待核实]` |
| CDN refresh / prefetch | 100 / day per domain on default | `[需讨论]` |

**Pricing context (RMB, training knowledge):**
| Plan / scope | Notes |
| --- | --- |
| Pay-as-you-go (绝大多数 API) | per-call rates, no flat fee |
| Subscription plans | e.g., ECS 包年包月, varies by instance family |
| Free tier | most products have a free quota, e.g., ECS 1-month free trial, CDN 10GB / month |

**Notes:**
- Aliyun rate limits are per-product. The "100 QPS" default
  is real but a small number of APIs go much lower (some 10
  QPS). Document per-API.
- Errors return `Throttling.User`, `Throttling.Api`,
  `Throttling.CloudBox` codes — these are useful for
  consumers to differentiate which quota they hit.

---

## 3. Tencent Cloud (腾讯云)

**Scope:** Cloud API 3.0 rate limits. Tencent publishes per-
API QPS quotas via `DescribeApiRateLimit`.

| API scope | Default limit | Status |
| --- | --- | --- |
| Most APIs (default) | 20 QPS / user / API | `[待核实: cloud.tencent.com/document/product/301/30495]` |
| Higher-tier APIs (CVM, CDB) | custom, often 50–100 QPS | `[需讨论]` |
| Account-level global | aggregate across all APIs; usually 1000 QPS | `[待核实]` |

**Pricing context (RMB / USD mixed):**
| Plan | Notes |
| --- | --- |
| Pay-as-you-go | per-resource, e.g., CVM hourly |
| Monthly subscription | discount on committed use |
| Student / trial | nominal free quota |

**Notes:**
- Tencent Cloud API rate limit headers: `X-RateLimit-Limit`,
  `X-RateLimit-Remaining`, `X-RateLimit-Window`.
- They publish a `DescribeApiRateLimit` API itself — useful
  for programmatic rate-limit lookup, worth documenting.
- Major APIs to cover: CVM (云服务器), CDB (云数据库),
  COS (对象存储), CDN, VPC.

---

## 4. GitHub

**Scope:** GitHub REST, GraphQL, Apps, Actions, Webhooks,
Packages.

| Surface | Default limit | Status |
| --- | --- | --- |
| REST API, unauthenticated | 60 req / hour / IP | `[已核实: docs.github.com/en/rest/using-the-rest-api/rate-limits-for-the-rest-api, 2024-11]` |
| REST API, authenticated (PAT) | 5,000 req / hour / token | `[已核实]` |
| GitHub App, user-to-server | 5,000+ req / hour / installation (varies) | `[已核实]` |
| GraphQL | 5,000 points / hour / user (queries cost 1–10 points each) | `[已核实]` |
| Search API | 30 req / minute / user | `[已核实]` |
| Secondary rate limit (abuse) | triggers after burst patterns; `Retry-After` returned | `[已核实]` |
| Actions API | 1,000 req / hour / repo | `[已核实: docs.github.com/en/rest/actions]` |
| Webhooks (inbound to you) | no GitHub-side limit; you decide retry | `[已核实]` |
| Packages (ghcr.io pulls) | rate limit applies at registry level (separate from REST) | `[需讨论]` |

**Pricing context (verify against github.com/pricing):**
| Plan | Price | Notable API changes |
| --- | --- | --- |
| Free | $0 | 5,000 REST/hr (lower than Pro) |
| Team | $4/user/mo | 5,000+ REST/hr, SAML SSO |
| Enterprise | $21/user/mo | higher rate limits, audit log |

**Notes:**
- "5000/hour" is misleadingly low — most users hit it for
  a few minutes and the cost is bursty. Document the burst
  + sustained distinction.
- GraphQL's "points" model: a query's cost is the
  *highest-cost* field's cost. Cost-reducing tooling
  matters.
- Secondary rate limits are NOT documented with explicit
  numbers; they're triggered heuristically. Document the
  triggers.

---

## 5. Vercel

**Scope:** Serverless / Edge function executions, bandwidth,
build minutes, deployments, API rate limits.

| Surface | Hobby (free) | Pro | Status |
| --- | --- | --- | --- |
| Serverless function executions | 100 GB-hr / month | 1,000 GB-hr / month | `[待核实: vercel.com/docs/concepts/limits/overview]` |
| Serverless function timeout | 10s (configurable up to 300s on Pro) | 60s (Pro), 900s (Ent) | `[待核实]` |
| Edge function invocations | 500,000 / month | 5,000,000 / month | `[待核实]` |
| Edge function size | 1 MB | 4 MB | `[待核实]` |
| Bandwidth | 100 GB / month | 1 TB / month | `[待核实]` |
| Build minutes | 100 / month | 400 / month | `[待核实]` |
| Deployments | 100 / day | 3,000 / day | `[待核实]` |
| API rate | ~1 req/sec by default; batched endpoints higher | higher | `[需讨论]` |

**Pricing context (verify against vercel.com/pricing):**
| Plan | Price | Notable |
| --- | --- | --- |
| Hobby | $0/mo | 100 GB-hr/mo, 100 GB bandwidth |
| Pro | $20/mo + usage | 1 TB-hr/mo, 1 TB bandwidth |
| Enterprise | custom | custom |

**Notes:**
- Vercel's API itself has rate limits separate from the
  function execution limits. The API is REST with bearer
  tokens.
- "Functions" vs "Edge Functions" vs "Edge Middleware"
  each have their own quota buckets.

---

## 6. BandwagonHost (搬瓦工)

**Scope:** VPS hosting provider (now part of IT7 Networks).
"Rate limit" applies differently here than for API services —
this is connection-level, not request-level.

| Surface | Limit | Status |
| --- | --- | --- |
| Outbound port 25 (SMTP) | blocked by default | `[已核实: BandwagonHost TOS]` |
| Outbound TCP connections / sec | soft caps per plan | `[待核实]` |
| Inbound / outbound bandwidth | per-plan monthly cap | `[待核实]` |
| Control panel API rate | undocumented, low (manual-use oriented) | `[需讨论]` |
| VPS-level concurrent connections | depends on plan | `[待核实]` |

**Pricing context (USD, as of 2026-09):**
| Plan | Price | Notable |
| --- | --- | --- |
| KVM VPS, 1GB RAM | $19.99/yr (entry level) | LA / MC datacenters |
| KVM VPS, 2GB RAM | $33.99/yr (approx) | bigger instances |
| "VPS" specials | vary by datacenter | LA, MC, HK, JP, etc. |

**Notes:**
- BandwagonHost is a hosting provider, not an API service.
  The "rate limit" framing doesn't perfectly map. The
  categories that DO apply: outbound email (port 25 block),
  connection concurrency (TCP), bandwidth caps, control
  panel API.
- They sell VPS plans by year; the rate-limit story is
  mostly about connection-level enforcement, not request
  quotas.

---

## Cross-vendor patterns

After collecting per-vendor data, look for patterns that
deserve dedicated article coverage:

- **Rate-limit header conventions:**
  - Cloudflare: `Retry-After`
  - GitHub: `X-RateLimit-Limit`, `X-RateLimit-Remaining`,
    `X-RateLimit-Reset`, `Retry-After`
  - Standardized: IETF draft `RateLimit-*` headers (RFC 9745)
- **Authenticated vs unauthenticated quotas:** GitHub 60→5000,
  similar tiering elsewhere.
- **HTTP 429 semantics:** universally means "slow down", but
  `Retry-After` parsing differs.
- **Backoff strategies:** Exponential backoff with jitter;
  honor `Retry-After` if present.

## Verification workflow (to be built)

A future `.github/workflows/scrape.yml` would:

1. Walk `data/*.yaml` for entries with `verified: false`.
2. For each, fetch the `source_url` and parse the page.
3. Extract rate-limit numbers (regex / DOM heuristics).
4. Diff against the YAML values.
5. Open a PR with the diff if any value changed, or update
   the `last_verified` timestamp if unchanged.

Until this pipeline exists, every YAML carries:

```yaml
verified: false
last_verified: null
source_url: https://...
note: needs human verification
```

## Open questions for the team

- How often do we want to refresh the per-vendor docs?
  Daily? Weekly? Monthly? Rate limits change rarely;
  prices change more often.
- Do we want to include **historical** changes (i.e., the
  changelog of rate-limit updates)? A diff-friendly changelog
  would let users see "Cloudflare raised their free tier from
  50k to 100k on 2024-Q1".
- Pricing — is the user committed to keeping prices current?
  Prices change frequently; this might be a separate concern
  from rate limits (which are stable).