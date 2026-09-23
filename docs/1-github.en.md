---
x-title: GitHub rate limits — quick reference (5 limits + 5 download methods)
x-desc: Quick reference: GitHub's 5 rate limits (REST 5000/hr, GraphQL 5000 points/hr, Search 30/min, Actions 1000/hr, secondary heuristic) + 5 alternative download methods when you hit the limit (Releases API / HTML / archive / raw / CDN), where only Releases API counts against the API quota.
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

# GitHub rate limits — quick reference (5 limits + 5 download methods)

---

## 1. The 5 rate limits at a glance

| Limit | Auth | Cap | Window | Unit |
| --- | --- | --- | --- | --- |
| **Primary REST** | PAT (Personal Access Token) / OAuth / GitHub App | 5000 req | 1 hr | token / installation |
| **Primary REST** | None | 60 req | 1 hr | source IP |
| **GraphQL** | Any | 5000 points | 1 hr | token / installation (cost-based) |
| **Search** | Any | 30 req | 1 min | user |
| **Actions API** | Any | 1000 req | 1 hr | repository |
| **GITHUB_TOKEN** | `${{ secrets.GITHUB_TOKEN }}` | 1000 req | 1 hr | workflow run / repo |
| **Secondary** | — | Heuristic | — | abuse detection |

**Unit key**: PAT / OAuth / user-to-server is **per token** (more tokens = more budget). **PAT is what you generate at GitHub → Settings → Developer settings → Personal access tokens → Generate new token** — most scripts and curl calls use this. GitHub App is **per installation** (more installations = more aggregate budget) — GitHub Apps are third-party apps installed on a repo/org (e.g., Dependabot, CI integrations). `${{ secrets.GITHUB_TOKEN }}` is **per workflow run / repo** (each run gets its own auto-expiring token, 1000/hr/repo shared across all runs in the repo). Search / Actions / GraphQL / REST core are **separate buckets** — `X-RateLimit-Resource` header tells you which one.

**Detailed mechanic per limit**:

- **Primary REST** — PAT / OAuth / GitHub App user tokens all share this bucket (GitHub Apps count per installation). The key is **token or installation**, not GitHub account — 3 PATs = 3 independent budgets. GitHub Apps can [request higher quotas](https://docs.github.com/en/apps/creating-github-apps/setting-up-a-github-app/about-choosing-a-github-app), but 5000/hr is enough for typical workflows.

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

- **Search** — 30/min, separate bucket. Search is much more expensive than REST primary (every query redoes indexing + ranking). REST primary quota does **NOT** cover `/search/*` — separate bucket.

- **Actions API** — 1000/hr/repo. All endpoints under `/repos/<o>/<r>/actions/*` share this. Dashboard integrations that poll Actions heavily will hit it.

- **Secondary** — heuristic abuse detection, no published threshold. Triggered by:
  - Short bursts (even with primary headroom)
  - Concurrent in-flight requests
  - Repeated identical-content requests in a short window

  Returns 429 + `Retry-After`. **Looks identical to primary 429** — you can't tell from the response which bucket fired.

---

## 2. Hit the limit? Try alternative downloads — 5 methods

| Method | URL | Counts vs API quota? | Best for |
| --- | --- | --- | --- |
| **Releases API** | `api.github.com/...` | ✅ Yes (5000/hr auth) | List releases, find asset URLs |
| **HTML scraping** | `github.com/.../releases/...` | ⚠️ Yes (undocumented UI limit, ~hundreds/hr/IP) | One-shot fallback when API exhausted |
| **Archive tarball** | `codeload.github.com/.../tar.gz/refs/...` or `github.com/.../archive/.../tar.gz` | ❌ No (Fastly CDN) | Whole-repo snapshot at known ref (≤100 MB) |
| **Raw content** | `raw.githubusercontent.com/...` | ❌ No (Fastly CDN) | Single-file fetch by path |
| **CDN mirrors** | `cdn.jsdelivr.net/gh/...` / `cdn.statically.io/gh/...` / `gcore.jsdelivr.net/gh/...` | ❌ No (CDN-level) | Fallback when GitHub is slow / throttled / down |

**Core strategy**: 1 API call to list releases (the only step that counts against quota), download via CDN or raw. API quota costs 1 request, downloads unlimited.

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

`${{ secrets.GITHUB_TOKEN }}` is GitHub Actions' auto-provided token:

- Auto-created per workflow run; destroyed when the run ends.
- On by default, no setup required.
- **Quota: 1000 req/hr/repo** — shared across all Actions API endpoints in that repo.

**CI pitfall**: N workflows running concurrently in the same repo all share GITHUB_TOKEN. They share the **single 1000/hr/repo budget**. One runaway workflow (heavy polling) takes out the rest.

Fixes:

- **Use a PAT instead** — per-token budget isolates each workflow.
- **Use a GitHub App installation token** — per-installation isolation.
- **Cap concurrency + conditional requests** — see Section 4 above.

---

## Not covered here

- **GitHub Apps higher-quota request flow** — see [docs](https://docs.github.com/en/apps).
- **GraphQL field-level cost table** — see [GraphQL resource limits](https://docs.github.com/en/graphql/overview/resource-limitations).
- **Full `x eget` implementation** — see FAQ `eget-comprehensive-considerations`.

---

## Sources

- Primary REST: <https://docs.github.com/en/rest/using-the-rest-api/rate-limits-for-the-rest-api>
- GraphQL: <https://docs.github.com/en/graphql/overview/resource-limitations>
- Search: <https://docs.github.com/en/rest/search>
- Actions: <https://docs.github.com/en/rest/actions>
- Secondary: <https://github.blog/developer-skills/github/how-to-prevent-secondary-rate-limit-issues/>
- GitHub Apps auth: <https://docs.github.com/en/apps/creating-github-apps/authenticating-with-a-github-app/about-authentication-with-a-github-app>