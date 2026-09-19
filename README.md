# x-cmd/ratelimit — vendor API rate limits, referenced

A reference table of API rate limits and quota caps across
the services the team uses most. This README is for visitors
who land on the GitHub repo; the long-form articles and
per-vendor deep-dives live under `docs/` (served as the
canonical site at `x-cmd.com/ratelimit`).

## What this repo is

- **Quick reference for engineers.** You came here to look
  up "Cloudflare's per-user API quota" or "GitHub's secondary
  rate-limit triggers". The data is under `data/<vendor>.yaml`;
  the per-vendor articles are under `docs/`.
- **Refreshable from upstream.** Each YAML carries a
  `verified` flag and a `docs_source` URL. A future
  `.github/workflows/scrape.yml` job will re-pull official
  docs on a schedule, diff against the YAML, and open a PR
  when upstream numbers move.
- **Two-license.** Source code, articles, scripts under
  Apache 2.0 (`LICENSE`). The rate-limit data tables under
  `data/` under CC-BY-4.0 (`LICENSE-data`) — attribution
  required, commercial use OK.

## Vendors covered

| Vendor | Surface covered | Status | Article |
| --- | --- | --- | --- |
| Cloudflare | REST API + per-product HTTP caps | unverified | [`docs/1-cloudflare.md`](./docs/1-cloudflare.md) |
| 阿里云 (Aliyun) | OpenAPI per-product QPS | unverified | [`docs/2-aliyun.md`](./docs/2-aliyun.md) |
| 腾讯云 (Tencent Cloud) | Cloud API 3.0 rate limits | unverified | [`docs/3-tencent.md`](./docs/3-tencent.md) |
| GitHub | REST + GraphQL + Actions + secondary limits | verified 2024-11 | [`docs/4-github.md`](./docs/4-github.md) |
| Vercel | Function/edge quotas + REST API | unverified | [`docs/5-vercel.md`](./docs/5-vercel.md) |
| BandwagonHost | VPS-level port / bandwidth / connection caps | unverified | [`docs/6-bandwagonhost.md`](./docs/6-bandwagonhost.md) |

## At a glance

| Vendor | Primary API limit | Window | Headers |
| --- | --- | --- | --- |
| Cloudflare (REST) | 1200 req | 5 min / user | `Retry-After`, `cf-mitigated` |
| Aliyun (open API) | 100 QPS | 1 sec / user | custom `Code` field, not RFC 6585 |
| Tencent Cloud (API 3.0) | 20 QPS | 1 sec / user | `X-RateLimit-*`, `Retry-After` |
| GitHub REST (PAT) | 5000 req | 1 hr / token | `X-RateLimit-*`, `Retry-After` |
| GitHub REST (unauth) | 60 req | 1 hr / IP | `X-RateLimit-*`, `Retry-After` |
| GitHub GraphQL | 5000 points | 1 hr / token | `X-RateLimit-*`, `Retry-After` |
| GitHub Search | 30 req | 1 min / user | `X-RateLimit-*` |
| Vercel REST | 1 req | 1 sec / token | lowercase `ratelimit-*` (RFC 9745 style) |

> ⚠️ Numbers above are subject to verification. Each
> `data/<vendor>.yaml` carries a `verified` flag and the
> official-docs URL; the CI scraper will refresh them
> against upstream on every release.

## Where to look next

- **`data/<vendor>.yaml`** — machine-readable, the source
  of truth.
- **`docs/0-ratelimit-overview.md`** — the canonical
  "how to use this repo" article.
- **`docs/<n>-<vendor>.md`** — per-vendor deep dive.
- **`RATELIMIT-RESEARCH.md`** — working notes, including
  what the team has verified vs. what's still pending.

## License

- `LICENSE` — Apache 2.0 (code, prose, scripts).
- `LICENSE-data` — CC-BY-4.0 (data files under `data/`).

## Contributing

See [`CONTRIBUTING.md`](./CONTRIBUTING.md). External PRs are
welcome for data corrections, new vendor files, and article
edits; the team signs off on every change.