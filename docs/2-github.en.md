---
x-title: GitHub rate limits — 5000/hr token + secondary heuristic + 5 download methods
x-desc: "GitHub's 5 rate limits: authenticated REST 5000/hr per token / Search 30/min / Actions 1000/hr per repo / GITHUB_TOKEN 1000/hr per repo / secondary heuristic. 5 alternative download methods when you hit the limit: Releases API / HTML / archive / raw / CDN (only Releases API counts against the API quota)."
x-sidebar: GitHub rate limits
x-keywords: github, rate limit, ratelimit, api quota, rest api, graphql, actions, secondary rate limit, x-ratelimit, pat, github app, releases api, codeload, raw.githubusercontent.com, jsdelivr
x-json-ld:
  '@context': https://schema.org
  '@graph':
    - '@type': TechArticle
      headline: 'GitHub rate limits'
      inLanguage: 'en'
      about: 'GitHub API rate limits across REST, GraphQL, Search, Actions, and secondary'
---

# GitHub rate limits — quick reference (5 limits + 5 download methods)

---

## 1. The 5 rate limits at a glance

| Limit | Auth | Cap | Window |
| --- | --- | --- | --- |
| **Primary REST** | PAT (Personal Access Token) / OAuth / GitHub App | 5000 req / token or installation | 1 hr |
| **Primary REST** | <code v-pre>${{ secrets.GITHUB_TOKEN }}</code> (Actions auto-token) | 1000 req / repo (shared by all workflow runs in the repo) | 1 hr |
| **Primary REST** | None | 60 req / source IP | 1 hr |
| **GraphQL** | PAT / OAuth / GitHub App (any one) | 5000 points / token (cost-based) | 1 hr |
| **Search** | PAT / OAuth / GitHub App (any one) | 30 req / user | 1 min |
| **Actions API** | PAT / OAuth / GitHub App (any one) | 1000 req / repo | 1 hr |
| **Secondary** | — | Heuristic trigger | — |

**Unit key** (read `/ X` part of the Cap column):
- `token`: each token = independent budget. 3 PATs = 3 independent 5000/hr buckets. **PAT is what you generate at GitHub → Settings → Developer settings → Personal access tokens → Generate new token** — most scripts and curl calls use this.
- `installation`: each GitHub App install = independent budget. GitHub Apps are third-party apps installed on a repo/org (e.g., Dependabot, CI integrations).
- `repo` (GITHUB_TOKEN): each repo = 1 budget, **all workflow runs in that repo share** — one runaway workflow takes out the rest.
- `user`, `source IP`: bucket by client identifier.
- REST, GraphQL, Search, Actions are 4 separate buckets (don't compete). `X-RateLimit-Resource` header tells you which one.

**Detailed mechanic per limit**:

- **Primary REST** — PAT / OAuth / GitHub App user tokens all share this bucket (GitHub Apps count per installation). The key is **token or installation**, not GitHub account — 3 PATs = 3 independent budgets. GitHub Apps can [request higher quotas](https://docs.github.com/en/apps/creating-github-apps/setting-up-a-github-app/about-choosing-a-github-app), but 5000/hr is enough for typical workflows.
- **Primary REST (Actions auto-token)** — <code v-pre>${{ secrets.GITHUB_TOKEN }}</code> is GitHub's auto-issued token for workflow runs. It's *also* Primary REST auth, but bucketed by **repo** at 1000/hr, so every workflow run in the repo competes for the same pool. See Section 5 for the CI-wall story and the permissions example.

- **GraphQL** — each query costs 1-10 points depending on the **highest-cost field** in the query:

  ```graphql
  query {
    repository(name: "x-cmd", owner: "x-cmd") {
      issues(first: 10) {       # 1 point
        nodes {
          comments(first: 100)  # 10 points (max)
        }
      }
    }
  }
  ```

  The query above costs **10 points** (the max field cost). Connection fields and aggregation fields cost more than simple field reads.

  **Inspect cost**: add `rateLimit { cost remaining resetAt }` field to your query.

  **Mix REST + GraphQL to spread load** — `X-RateLimit-Resource` exposes `graphql` (5000 points/hr) and `core` (5000 req/hr) as **separate budgets**. Same token, two different buckets — you don't have to pick one. A practical pattern: use GraphQL to gather N related fields in one shot (cheaper in round-trips and easier to bound total cost), use REST for one-off lookups where shaping a GraphQL query isn't worth it.

  **Aggregation features are dramatically cheaper via GraphQL** — when one screen needs star count, release info, contributor count, languages, and a "merged PRs in last 30d" all together, that's a 5-call REST sequence (or worse, 8+ REST search calls once you add time windows). One GraphQL query fetches the same data and bills against the 5000 points/hr bucket instead of mixing 30/min search with 5000/h core. Concrete case: `x repo card` went from **11 HTTP requests per card at 30/min search bottleneck (~180 cards/hr)** to **3 HTTP requests per card at 5000/h graphql bottleneck (~2500 cards/hr)** — a **~14× throughput** lift, just by moving the search aggregations into a single GraphQL root-level `search(...)` aliased next to `repository(...)`. Story: [`x-bash/repo/.x-cmd/story/260824.x-repo-card-graphql-consolidation.md`](https://github.com/x-bash/repo/blob/main/.x-cmd/story/260824.x-repo-card-graphql-consolidation.md).

- **Search** — 30/min, separate bucket. Search is much more expensive than REST primary (every query redoes indexing + ranking). REST primary quota does **NOT** cover `/search/*` — separate bucket.

- **Actions API** — 1000/hr/repo. All endpoints under `/repos/<o>/<r>/actions/*` share this. Dashboard integrations that poll Actions heavily will hit it.

- **Secondary** — an abuse-detection layer sitting on top of the primary quota. No published thresholds; GitHub fires it when it sees:
  - **Burst traffic** — a flood of requests in a short window, even if your primary quota still has plenty of room
  - **Concurrent pile-up** — too many requests in flight at the same time
  - **Repeat hammering** — asking for the same resource over and over within seconds

  Response is 429 + `Retry-After`, **indistinguishable from a primary 429**. There's no header that says "this is secondary"; you only know which bucket fired by checking your own traffic pattern afterward.

---

## 2. Hit the limit? Try alternative downloads — 5 methods

| Method | URL | Counts vs API quota? | Best for |
| --- | --- | --- | --- |
| **Releases API** | `api.github.com/...` | ✅ Yes (5000/hr auth) | List releases, find asset URLs |
| **HTML scraping** | `github.com/.../releases/...` | ⚠️ Yes (undocumented UI limit, ~hundreds/hr/IP) | One-shot fallback when API exhausted |
| **Archive tarball** | `codeload.github.com/.../tar.gz/refs/...` or `github.com/.../archive/.../tar.gz` | ❌ No (Fastly CDN) | Whole-repo snapshot at known ref (≤100 MB) |
| **Raw content** | `raw.githubusercontent.com/...` | ❌ No (Fastly CDN) | Single-file fetch by path |
| **jsDelivr CDN mirror** | `cdn.jsdelivr.net/gh/...` (also `cdn.statically.io/gh/...` / `gcore.jsdelivr.net/gh/...`) | ❌ No (CDN-level) | **Hit the limit / GitHub slow / down — first choice** |

**Core strategy**: 1 API call to list releases (the only step that counts against quota), download via CDN or raw. API quota costs 1 request, downloads unlimited.

**First-choice CDN = jsDelivr** — not just a raw mirror. Also supports semver (`@1` / `@^1.2`), file combining, npm packages, multi-CDN fallback (`gcore.jsdelivr.net`), ~50M requests/month free tier. Full capabilities in FAQ [`jsdelivr-github-capabilities`](#).

```sh
# Step 1: list releases (1 API call)
curl -H "Authorization: Bearer $TOKEN" \
     'https://api.github.com/repos/x-cmd/x-cmd/releases?per_page=100'

# Step 2: download (CDN, no API cost)
curl -L -o release.tar.gz \
     https://github.com/x-cmd/x-cmd/archive/refs/tags/v1.0.0.tar.gz

# Step 3: single file (CDN, no API cost)
curl -L -o README.md \
     https://raw.githubusercontent.com/x-cmd/x-cmd/v1.0.0/README.md
```

---

## 3. Key response headers

| Header | Meaning | When |
| --- | --- | --- |
| `X-RateLimit-Limit` | Current bucket total | Every response |
| `X-RateLimit-Remaining` | Remaining | Every response |
| `X-RateLimit-Reset` | **UNIX epoch timestamp** (NOT "N seconds left") | Every response |
| `X-RateLimit-Used` | Used | Every response |
| `X-RateLimit-Resource` | Current bucket (`core` / `search` / `graphql` etc.) | Every response |
| `Retry-After` | Integer seconds | On 429 |

**Footgun**: `X-RateLimit-Reset` is UNIX epoch (e.g., `1640000000`), not a relative value. To compute remaining time: `reset - now`. `Retry-After` IS relative seconds (only on 429).

---

## 4. 429 / 403 — how to respond

| Trigger | HTTP | Key header | Fix |
| --- | --- | --- | --- |
| Primary quota reached | 403 | `X-RateLimit-Remaining: 0` | Sleep until `X-RateLimit-Reset` |
| Secondary heuristic triggered | 429 | `Retry-After` | Honor `Retry-After` + change pattern |
| Search quota reached | 403 | `X-RateLimit-Resource: search` | Sleep 1 min + search less |

**Note**: Primary and Secondary 429 look identical — you can't tell from the response which bucket fired.

```python
import time
import requests

def call_github(url, headers, max_retries=5):
    for attempt in range(max_retries):
        r = requests.get(url, headers=headers)
        if r.status_code == 200:
            return r
        if r.status_code == 429:
            # Primary or secondary — both honor Retry-After
            time.sleep(int(r.headers.get("Retry-After", "60")))
            continue
        if r.status_code == 403:
            if r.headers.get("X-RateLimit-Remaining") == "0":
                # Primary quota reached — sleep until reset
                wait = max(int(r.headers["X-RateLimit-Reset"]) - time.time(), 1)
                time.sleep(min(wait, 3600))
                continue
        r.raise_for_status()
    raise RateLimitExceeded()
```

Points:

1. **Pre-flight check `X-RateLimit-Remaining`**. If 0, don't bother — sleep until reset.
2. **Honor `Retry-After`**. Both primary and secondary set it.
3. **Track per-token budget locally**. `X-RateLimit-Remaining` is authoritative, but don't burn a request to read it — store a local counter and decrement.
4. **Avoid bursts**. Parallel ≤ 5-10, otherwise secondary fires.
5. **Use conditional requests**. `If-None-Match` / `If-Modified-Since` get 304 without consuming quota.
6. **CDN whenever possible**. List via API (1 call), download via archive / raw / CDN (free).

---

## 5. GITHUB_TOKEN — the CI wall

<code v-pre>${{ secrets.GITHUB_TOKEN }}</code> is GitHub Actions' auto-provided token:

- Auto-created per workflow run; destroyed when the run ends.
- On by default, no setup required.
- **Quota: 1000 req/hr/repo** — shared across all Actions API endpoints in that repo.

**Declared permissions** (not a rate-limit knob, but adjacent — the token's reach is exactly what you whitelist):

```yaml
# .github/workflows/ci.yml
name: ci
on: [push, pull_request]

permissions:
  contents: read        # checkout the repo
  issues: write         # open / comment on issues
  pull-requests: write  # comment / label PRs
  checks: write         # write check runs

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: |
          curl -H "Authorization: Bearer ${{ secrets.GITHUB_TOKEN }}" \
               https://api.github.com/repos/${{ github.repository }}/issues
```

Default permission set is `contents: read` at the workflow level since 2023 — anything else you have to opt into. The token never has write access to anything outside what you list under `permissions:`, even if the repo's default branch allows it.

**CI pitfall**: N workflows running concurrently in the same repo all share GITHUB_TOKEN. They share the **single 1000/hr/repo budget**. One runaway workflow (heavy polling) takes out the rest.

Fixes:

- **Use a PAT instead** — per-token budget isolates each workflow.
- **Use a GitHub App installation token** — per-installation isolation.
- **Cap concurrency + conditional requests** — see Section 4 above.

---


---

## Sources — based on the GitHub blog

- Secondary rate limits (the canonical engineering write-up — also covers primary quota, Actions, and how the layers interact): <https://github.blog/developer-skills/github/how-to-prevent-secondary-rate-limit-issues/>
