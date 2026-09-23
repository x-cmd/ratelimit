---
x-title: GitHub — rate limits and how to avoid them with alternative surfaces
x-desc: GitHub's primary rate limits (5000/hr authenticated REST, 60/hr unauthenticated, 30/min search, 5000 points/hr GraphQL, 1000/hr Actions), secondary rate-limit triggers and headers (X-RateLimit-*). Plus a 4-row cheatsheet on the five download surfaces (Releases API / HTML / archive tarball / raw / CDN) and which count against the API budget.
x-sidebar: GitHub rate limits + alt surfaces
x-keywords: github, ratelimit, api quota, rest api, graphql, actions, secondary rate limit, x-ratelimit, oauth, github app, releases api, codeload, raw.githubusercontent.com, jsdelivr
x-json-ld:
  '@context': https://schema.org
  '@graph':
    - '@type': TechArticle
      headline: 'GitHub rate limits'
      inLanguage: 'en'
      about: 'GitHub API rate limits across REST, GraphQL, Search, Actions, and secondary'
---

# GitHub rate limits

GitHub has the most-publicly-documented rate-limit story
of any major API provider. Five surfaces matter:

1. **Primary REST** — 5000/hr authenticated, 60/hr unauthenticated.
2. **GraphQL** — 5000 points/hour, cost-based.
3. **Search** — 30/min, separate from REST primary.
4. **Actions API** — 1000/hr/repo.
5. **Secondary** — heuristic abuse-detection rate limit.

Plus five **download surfaces** (Releases API, HTML scraping,
archive tarball, raw content, CDN mirrors) — only one of
which counts against the GitHub API budget. See the cheatsheet
near the end of this article, and the FAQ for the why and
how (eget strategy, gitattributes, cloud dev envs).

---

## Primary REST API rate limits

| Auth surface | Limit | Window | Per |
| --- | --- | --- | --- |
| Unauthenticated | 60 req | 1 hr | source IP |
| Personal access token (PAT) | 5,000 req | 1 hr | token |
| OAuth app (user token) | 5,000 req | 1 hr | token |
| GitHub App user-to-server | 5,000+ req | 1 hr | installation |
| GitHub App server-to-server | 5,000+ req | 1 hr | installation |

The 5,000/hr ceiling is the same across OAuth / PAT /
user-to-server. **GitHub Apps can request higher quotas**
via the `increasing-api-quota-for-github-apps` request
form, but for typical workflows 5,000/hr is enough.

### What "per token" means for OAuth / PAT

The quota is **per token**, not per user. A user with three
PATs gets three independent 5,000/hr quotas. Useful for CI
isolation: one PAT per workflow, one budget per workflow.

### GitHub Apps: per-installation quota

GitHub Apps have **per-installation** quotas — the 5,000/hr
counts against the installation, not against the app
overall. If your app is installed on 100 installations, you
effectively get up to 500,000/hr aggregated, but each
installation's quota is tracked separately.

## GraphQL API

GraphQL uses a **cost-based** quota: 5,000 points per hour
per token-or-installation. Each query consumes 1–10
points depending on the *highest-cost* field in the query.

```graphql
query {
  repository(name: "x-cmd", owner: "x-cmd") {
    issues(first: 10) {       # costs 1 point
      nodes {
        comments(first: 100) # costs 10 points (max)
      }
    }
  }
}
```

The query above costs 10 points (the max of any field).
Connection fields and aggregation fields cost more than
simple field reads.

**Pro tip**: ask for `cost { totalCost }` in your query to
inspect actual consumption:

```graphql
query {
  repository(...) { ... }
  rateLimit {
    limit
    cost
    remaining
    resetAt
  }
}
```

## Search API

The `/search/*` endpoints have a separate, lower quota:

| Surface | Limit | Window | Per |
| --- | --- | --- | --- |
| Search API (any auth) | 30 req | 1 min | user |

30/min is much lower than REST primary because search is
expensive (indexing, ranking). The REST primary 5,000/hr
quota does **NOT** cover `/search/*` — they're separate
budgets.

## Actions API

| Surface | Limit | Window | Per |
| --- | --- | --- | --- |
| REST API under `/repos/{owner}/{repo}/actions/*` | 1,000 req | 1 hr | repository |

Workflow artifact downloads, list-runs, and other
Actions-related REST endpoints share this 1,000/hr/repo
quota. CI tooling that polls Actions heavily (e.g.,
dashboard integrations) can hit this.

## Secondary rate limit (the surprise one)

GitHub enforces a *secondary* rate limit on top of the
primary one. It's heuristic — triggered by abuse-like
patterns:

- Too many requests in a short burst (regardless of primary
  quota headroom).
- Concurrent requests in flight.
- Repeated requests that return the same content within a
  short window.

When triggered, GitHub returns:

```http
HTTP/1.1 429 Too Many Requests
Retry-After: 60
X-RateLimit-Reset: 1640000000
```

There's no documented "you have N secondary requests
remaining" header. The trigger conditions are described
in a [GitHub blog post](https://github.blog/developer-skills/github/how-to-prevent-secondary-rate-limit-issues/)
but the exact thresholds aren't public.

### Avoiding secondary limits

- **Avoid bursts.** Even with 4,000/hr primary headroom,
  firing 100 requests in parallel can trigger secondary.
- **Conditional requests.** Use `If-None-Match` /
  `If-Modified-Since` headers — GitHub returns 304 without
  consuming API quota for unchanged resources.
- **Avoid tight polling.** Don't poll every second; back off
  exponentially.

## Headers GitHub emits

Standard headers across REST + GraphQL:

```http
X-RateLimit-Limit: 5000
X-RateLimit-Remaining: 4999
X-RateLimit-Reset: 1640000000
X-RateLimit-Used: 1
X-RateLimit-Resource: core    # "core", "search", "graphql", etc.
Retry-After: 60                # only on 429
```

**`X-RateLimit-Reset` is a UNIX epoch timestamp**, not
seconds-until-reset. This trips up first-time consumers.

`X-RateLimit-Resource` lets you know which bucket you're
being tracked against — important when a single client
uses both `/search/*` (30/min) and core REST (5,000/hr).

## Client-side retry strategy

A robust GitHub client:

```python
import time
import requests

def call_github(url, headers, max_retries=5):
    for attempt in range(max_retries):
        response = requests.get(url, headers=headers)
        if response.status_code == 200:
            return response
        if response.status_code == 429:
            retry_after = int(response.headers.get("Retry-After", "60"))
            time.sleep(retry_after)
            continue
        if response.status_code == 403:
            if response.headers.get("X-RateLimit-Remaining") == "0":
                reset_at = int(response.headers["X-RateLimit-Reset"])
                wait = max(reset_at - time.time(), 1)
                time.sleep(min(wait, 3600))
                continue
        response.raise_for_status()
    raise RateLimitExceeded()
```

Three rules of thumb:

1. **Pre-flight check `X-RateLimit-Remaining`** before each
   request. If it's 0, don't bother — sleep until reset.
2. **Respect `Retry-After`** when present. Secondary rate
   limits set it; primary does too on 429.
3. **Track per-token budget locally.** `X-RateLimit-Remaining`
   is authoritative but you don't need to make a request
   just to find out — store the counter locally and
   decrement on each request.

---

## Download surfaces — cheatsheet

The same `x-cmd/x-cmd` release artifact is reachable
through **five different URLs**, each with its own
rate-limit story. The full strategy and trade-offs live in
the FAQ (`github-five-download-surfaces`).

| Surface | URL | Counts vs API budget? | Best for |
| --- | --- | --- | --- |
| **Releases API** | `api.github.com/...` | ✅ Yes (5000/hr auth / 60/hr unauth) | Listing releases, finding asset URLs |
| **HTML scraping** | `github.com/.../releases/...` | ⚠️ Yes — undocumented UI limit ~hundreds/hr per IP | One-shot fallback when API exhausted |
| **Archive tarball** | `codeload.github.com/.../tar.gz/refs/...` or `github.com/.../archive/.../tar.gz` | ❌ No (Fastly CDN) | Whole-repo snapshot at a known ref (≤100 MB cap) |
| **Raw content** | `raw.githubusercontent.com/...` | ❌ No (Fastly CDN) | Single-file fetch by path |
| **CDN mirrors** | `cdn.jsdelivr.net/gh/...`, `cdn.statically.io/gh/...`, `gcore.jsdelivr.net/gh/...` | ❌ No (CDN-level) | Fallback when GitHub is slow / throttled / unavailable |

**Key insight**: only the Releases API counts. Use the
API for listing (1 request to discover all releases), then
use `codeload.github.com` or `raw.githubusercontent.com` to
download — both are CDN-backed and exempt from the API
budget.

**Strategy cheatsheet** — for a tool that wants every release
of `x-cmd/x-cmd`:

```sh
# Step 1: list releases (1 API call)
curl -H "Authorization: Bearer $TOKEN" \
     'https://api.github.com/repos/x-cmd/x-cmd/releases?per_page=100'

# Step 2: download each release (CDN, no API cost)
curl -L -o release.tar.gz \
     https://github.com/x-cmd/x-cmd/archive/refs/tags/v1.0.0.tar.gz

# Step 3: fetch a single file (CDN, no API cost)
curl -L -o README.md \
     https://raw.githubusercontent.com/x-cmd/x-cmd/v1.0.0/README.md
```

API budget: 1 request. CDN downloads: unbounded.

---

## Sources

- Primary REST rate limits:
  <https://docs.github.com/en/rest/using-the-rest-api/rate-limits-for-the-rest-api>
- GraphQL resource limits:
  <https://docs.github.com/en/graphql/overview/resource-limitations>
- Search API rate limits:
  <https://docs.github.com/en/rest/search>
- Actions API:
  <https://docs.github.com/en/rest/actions>
- Secondary rate-limit blog post (trigger patterns):
  <https://github.blog/developer-skills/github/how-to-prevent-secondary-rate-limit-issues/>
- GitHub Apps auth model:
  <https://docs.github.com/en/apps/creating-github-apps/authenticating-with-a-github-app/about-authentication-with-a-github-app>

**Verification status**: verified 2024-11 against the above.