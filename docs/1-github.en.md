---
x-title: GitHub — rate limits and download strategies (releases, HTML, archive, raw, CDN mirrors)
x-desc: GitHub's primary rate limits (5000/hr authenticated REST, 60/hr unauthenticated, 30/min search, 5000 points/hr GraphQL, 1000/hr Actions), secondary rate-limit triggers, plus a deep dive on five download strategies: Releases API, HTML scraping, archive/tarball, raw content, and CDN mirrors — and how they interact with rate limits.
x-sidebar: GitHub rate limits + downloads
x-keywords: github, ratelimit, api quota, rest api, graphql, actions, secondary rate limit, x-ratelimit, oauth, github app, releases api, codeload, raw.githubusercontent.com, jsdelivr, gitattributes, sparse-checkout, colab, codespaces
x-json-ld:
  '@context': https://schema.org
  '@graph':
    - '@type': TechArticle
      headline: 'GitHub rate limits and download strategies'
      inLanguage: 'en'
      about: 'GitHub API rate limits and the five download strategies'
---

# GitHub — rate limits and download strategies

Two stories in this article:

1. **GitHub API rate limits** — primary REST, GraphQL, Search, Actions, secondary.
2. **GitHub download strategies** — five ways to fetch code/release artifacts, and how each interacts with rate limits.

The second story is the one that catches developers off-guard:
GitHub exposes the same content through **five distinct surfaces**,
each with its own rate-limit semantics. If you've been hitting
60 req/hr (the unauthenticated quota) and wondered why — you
might have been using the API when the right surface is
**raw.githubusercontent.com** (no rate limit) or **archive
tarballs** (CDN-cached, no rate limit).

---

## Why GitHub?

GitHub is the upstream source for nearly every developer
tool. Three traffic patterns matter:

- **Browsing** — `github.com/<owner>/<repo>` — UI rate
  limits, undocumented but enforced (~hundreds/hr).
- **API** — `api.github.com/...` — documented primary rate
  limits (5,000/hr authenticated).
- **Raw / archive / CDN** — `raw.githubusercontent.com` and
  `codeload.github.com` — CDN-cached, **no documented rate
  limit** (soft limits apply).

If you're scraping GitHub at scale, picking the right
surface is more important than authenticating.

---

## GitHub download strategies — five ways to fetch

The same `x-cmd/x-cmd` release artifact is reachable
through **five different URLs**, each with its own
rate-limit story.

### Surface 1 — Releases API (`api.github.com`)

The canonical programmatic way to list releases and
download release assets.

```sh
# List releases
curl -H "Accept: application/vnd.github+json" \
     https://api.github.com/repos/x-cmd/x-cmd/releases

# Download a specific release asset
curl -L \
     -H "Authorization: Bearer $TOKEN" \
     -o x-cmd.tar.gz \
     https://api.github.com/repos/x-cmd/x-cmd/releases/tags/v1.0.0
```

| Property | Value |
| --- | --- |
| Rate limit | **5,000/hr auth, 60/hr unauth** (GitHub primary REST) |
| Limits | Listed releases (default 30), but paginated |
| Best for | "What version is current?" — list, filter, find asset URL |
| Cost | Each call counts 1 against your REST budget |

**Limits**:

- **60 req/hr unauthenticated** — the most-cited "GitHub is
  rate-limiting me" complaint. Often the right answer is to
  add a token; sometimes the right answer is to use a
  different surface (see below).
- **Secondary rate limit** — burst patterns trigger
  heuristic throttling even with budget remaining. See the
  Secondary Rate Limit section below.

### Surface 2 — HTML scraping (`github.com/.../releases/tag/...`)

When you don't have a token, the obvious fallback is to
scrape the HTML release page.

```sh
curl -L -A "Mozilla/5.0" \
     https://github.com/x-cmd/x-cmd/releases/tag/v1.0.0 \
     | grep -oP 'href="[^"]*\.(tar\.gz|zip)"'
```

| Property | Value |
| --- | --- |
| Rate limit | **Undocumented UI limit** (~hundreds/hr per IP) |
| Limits | Brittle — page structure changes silently break scrapers |
| Best for | One-off manual download; fallback when API quota is exhausted |
| Cost | Server-side rate limit; no per-token budget |

**Limits**:

- **Undocumented threshold** — GitHub doesn't publish
  numbers. Long-term scrapers will be 429'd.
- **No personal budget** — the limit is per IP, not per
  token. Behind a NAT, your whole office shares one bucket.
- **DOM drift** — the next CSS class rename breaks your
  regex.
- **Headless detection** — GitHub serves different HTML to
  "browser-like" User-Agents vs. plain `curl` UA. Set the
  UA and hope.

Use HTML scraping only when the API quota is genuinely
exhausted and you need a one-shot pull. Not for production.

### Surface 3 — Archive / tarball (entire repo at a ref)

The biggest GitHub-rate-limit escape hatch: **download the
entire repo as a tarball**. Served by Fastly CDN; not the
GitHub API.

```sh
# Branch tarball
curl -L -o x-cmd-main.tar.gz \
     https://codeload.github.com/x-cmd/x-cmd/tar.gz/refs/heads/main

# Tag tarball
curl -L -o x-cmd-v1.0.0.tar.gz \
     https://codeload.github.com/x-cmd/x-cmd/tar.gz/refs/tags/v1.0.0

# Or via github.com URL (same CDN)
curl -L -o x-cmd-v1.0.0.tar.gz \
     https://github.com/x-cmd/x-cmd/archive/refs/tags/v1.0.0.tar.gz
```

| Property | Value |
| --- | --- |
| Rate limit | **CDN-only (Fastly); no documented GitHub API rate limit** |
| Size cap | ~100 MB compressed per archive (GitHub's hard cap) |
| Best for | **Fetching the entire source of a repo at a known ref** |
| Cost | Network bandwidth, not API quota |

**Limits**:

- **Repo size cap** — GitHub enforces a 100 MB compressed
  cap on archives. Larger repos can't be downloaded as a
  single tarball. Use git clone instead.
- **LFS files are pointers, not blobs.** Archive tarballs
  don't contain actual LFS content. Download LFS separately.
- **Submodules** — archive tarballs don't include
  submodules. For repos with submodules, git clone is the
  only full-fidelity option.

This is the surface that **saves most bots** — the
release-list API quota is exhausted by listing all releases,
but the tarball is a single CDN request with no API
budget cost.

### Surface 4 — Raw content (`raw.githubusercontent.com`)

The simplest escape hatch: **fetch one file at a time
through GitHub's CDN**. No API, no HTML scraping, no
tarball expansion.

```sh
# Single file
curl -L https://raw.githubusercontent.com/x-cmd/x-cmd/main/README.md

# Specific tag
curl -L https://raw.githubusercontent.com/x-cmd/x-cmd/v1.0.0/INSTALL.md

# Sub-directory tree (use raw URLs per-file, not recursive)
for f in install.d/01.sh install.d/02.sh install.d/03.sh; do
  curl -L "https://raw.githubusercontent.com/x-cmd/x-cmd/main/$f"
done
```

| Property | Value |
| --- | --- |
| Rate limit | **CDN-only (Fastly); no documented GitHub API rate limit** |
| Soft limit | ~60-100 req/min per source IP (observed; not published) |
| Best for | **Files you know the path of** — manifests, configs, single scripts |
| Cost | Network bandwidth, not API quota |

**Why `raw.xxx` bypasses ratelimit**: GitHub's API rate
limit applies to `api.github.com` and `github.com` HTML
pages. `raw.githubusercontent.com` is a separate CDN fronted
by Fastly — the same CDN that serves `cdn.jsdelivr.net` and
GitHub Releases downloads. You are not consuming API budget.

**Limits**:

- **Soft IP-based rate limit** — there IS one, just not
  documented. Heavy automation (e.g., fetching thousands of
  files per minute from one IP) gets throttled. The exact
  threshold is observed at ~60-100 req/min.
- **No recursive directory listing** — `raw.githubusercontent.com`
  doesn't enumerate; you need to know the path. For a
  recursive download, use `git clone` or the tarball.
- **No auth** — tokens don't help. `raw.xxx` is anonymous
  by design.

### Surface 5 — CDN mirrors (jsDelivr, Statically, gcore)

Third-party CDNs that mirror GitHub releases and raw
content. Useful when GitHub itself is throttled or
geoblocked.

```sh
# jsDelivr (most common)
curl -L https://cdn.jsdelivr.net/gh/x-cmd/x-cmd@main/README.md
curl -L https://cdn.jsdelivr.net/gh/x-cmd/x-cmd@v1.0.0/INSTALL.md

# Statically
curl -L https://cdn.statically.io/gh/x-cmd/x-cmd/main/README.md

# gcore
curl -L https://gcore.jsdelivr.net/gh/x-cmd/x-cmd@main/README.md
```

| Property | Value |
| --- | --- |
| Rate limit | **CDN-level (no per-user limit; ~50M req/month on jsDelivr free tier)** |
| Coverage | Tag, branch, commit, semver ranges (`@1`, `@^1.2`) |
| Best for | **Fallback when GitHub is slow / throttled / unavailable** |
| Cost | Free tier has usage caps; paid tiers available |

**Limits**:

- **Eventually consistent** — CDN propagation takes seconds
  to minutes after a new release. Don't use for time-
  sensitive freshness.
- **No private repos** — public repos only.
- **Size limits per file** — typically 50 MB per file on
  jsDelivr (similar to GitHub).
- **CSP / SRI friction** — if you're loading from jsDelivr,
  you need to add `cdn.jsdelivr.net` to your CSP and use
  SRI hashes for security.

### Surface comparison — the four-limits matrix

| Surface | URL | Rate limit | Soft limit | Best for |
| --- | --- | --- | --- | --- |
| **Releases API** | `api.github.com/...` | 5000/hr auth, 60/hr unauth | Burst → secondary | Listing releases |
| **HTML scraping** | `github.com/.../releases/...` | ~hundreds/hr per IP | DOM drift | One-shot fallback |
| **Archive tarball** | `codeload.github.com/.../tar.gz/refs/...` | **CDN (Fastly) — no documented GitHub limit** | Repo size ≤100 MB | Whole-repo snapshot |
| **Raw content** | `raw.githubusercontent.com/...` | **CDN (Fastly) — no documented GitHub limit** | ~60-100 req/min per IP | Single-file fetch |
| **CDN mirrors** | `cdn.jsdelivr.net/gh/...` | CDN-level (~50M req/month free) | Eventually consistent | Fallback / CDN |

### A bot's path through the five surfaces

For a tool that wants to fetch every release of `x-cmd/x-cmd`:

```sh
# Step 1: Releases API — list all releases (1 API call per page)
curl -H "Authorization: Bearer $TOKEN" \
     'https://api.github.com/repos/x-cmd/x-cmd/releases?per_page=100&page=1'

# Step 2: For each release, fetch its tarball via archive (CDN, no API cost)
curl -L -o release.tar.gz \
     https://github.com/x-cmd/x-cmd/archive/refs/tags/v1.0.0.tar.gz

# Step 3: For specific files inside the repo, use raw (CDN, no API cost)
curl -L -o README.md \
     https://raw.githubusercontent.com/x-cmd/x-cmd/v1.0.0/README.md
```

API budget: 1 request to list. CDN downloads: unbounded.

**vs HTML scraping**: 5 lines × every release × every ref.
Brittle, undocumented limit, no auth, IP-bucketed.

### The fourth angle: rate limit when fetching these three

You might ask: *"if I use the Releases API to list, then the
archive for the tarball, then raw for a single file — how do
rate limits interact?"*

Answer: **only the Releases API counts**. The archive and
raw surfaces are CDN-cached and do not consume GitHub API
quota. So:

| Call | Counts against GitHub primary? |
| --- | --- |
| `api.github.com/...` | ✅ Yes (5000/hr auth or 60/hr unauth) |
| `github.com/.../archive/...` (tarball) | ❌ No |
| `raw.githubusercontent.com/...` | ❌ No |
| `cdn.jsdelivr.net/gh/...` | ❌ No |
| `github.com/.../releases/...` (HTML) | ⚠️ Yes — undocumented UI limit |

Strategy: **let the Releases API do the listing, let
`raw.xxx` and `codeload.github.com` do the downloading**.

### Developer angle — gitattributes + cloud dev environments

If your target is **the project itself as code** (not a
release asset), and you're worried about repo size, submodules,
or LFS — don't use the tarball. Use git directly, with
**gitattributes** to trim what you pull.

```sh
# Sparse checkout — only one path
git clone --depth=1 --filter=blob:none --sparse \
     https://github.com/x-cmd/x-cmd.git
cd x-cmd
git sparse-checkout set install.d docs

# Or, use .gitattributes in your own repo to trim push size
# (so consumers get a smaller tarball too):
# .gitattributes
# install.d/*.tar.gz   filter=lfs diff=lfs merge=lfs -text
# large-dataset/*      filter=lfs diff=lfs merge=lfs -text
```

For **cloud dev environments** — i.e., "I don't even want to
download, I want a sandbox":

- **GitHub Codespaces** — official; browser-based; GH
  quotas apply.
- **Gitpod** — open-source alternative; works with any GH
  repo; free tier available.
- **Google Colab** — Python-first; clone via
  `!git clone https://github.com/x-cmd/x-cmd.git`. No
  release download needed.

These environments **stream the repo directly from
GitHub** without you pulling bytes — the network traffic
counts against GitHub's CDN, not your API quota.

### Reference: `x eget` story

The x-cmd [`eget`](https://github.com/x-bash/eget)
module implements this exact strategy:

1. Try the **Releases API** first to find the right
   asset URL (1 API call).
2. If API is exhausted, **fall back to HTML scraping** of
   `github.com/<o>/<r>/releases/tag/<tag>` to extract asset
   URLs (brittle but rate-limited separately).
3. Download the asset via **`codeload.github.com` or
   `raw.githubusercontent.com`** (CDN, no API quota).

The eget flow is documented in
[`x-bash/eget/lib/download`](https://github.com/x-bash/eget).
The key invariant: **only step 1 touches the API budget**.

---

## Primary REST API rate limits

The "GitHub rate limit" most developers hit.

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
PATs gets three independent 5,000/hr quotas. This is
useful for CI isolation: one PAT per workflow, one budget
per workflow.

### GitHub Apps: per-installation quota

GitHub Apps have **per-installation** quotas — the
5,000/hr counts against the installation, not against the
app overall. If your app is installed on 100 repositories
across 100 installations, you effectively get up to
500,000/hr aggregated, but each installation's quota is
tracked separately.

## GraphQL API

GraphQL uses a **cost-based** quota: 5,000 points per hour
per token-or-installation. Each query consumes 1–10
points depending on the *highest-cost* field in the
query.

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
Multipliers scale: connection fields and aggregation
fields cost more than simple field reads.

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

- Too many requests in a short burst (regardless of
  primary quota headroom).
- Concurrent requests in flight.
- Repeated requests that return the same content within
  a short window.

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
- **Avoid tight polling.** Don't poll every second; back
  off exponentially.
- **Pick the right surface.** Listing releases? API.
  Downloading? Archive tarball. Single file? Raw content.

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
            # Secondary rate limit; sleep Retry-After exactly.
            time.sleep(retry_after)
            continue
        if response.status_code == 403:
            # X-RateLimit-Remaining == 0 means quota exhausted.
            if response.headers.get("X-RateLimit-Remaining") == "0":
                reset_at = int(response.headers["X-RateLimit-Reset"])
                wait = max(reset_at - time.time(), 1)
                time.sleep(min(wait, 3600))
                continue
        # Non-rate-limit error; let caller handle.
        response.raise_for_status()
    raise RateLimitExceeded()
```

Three rules of thumb:

1. **Pre-flight check `X-RateLimit-Remaining` before each
   request.** If it's 0, don't bother — sleep until reset.
2. **Respect `Retry-After`** when present. Secondary rate
   limits set it; primary does too on 429.
3. **Pick the right surface.** Listing releases? Use the
   API. Downloading the tarball? Use `codeload.github.com`.
   Fetching a single file? Use `raw.githubusercontent.com`.

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
- codeload.github.com (archive tarballs):
  <https://docs.github.com/en/repositories/working-with-files/using-files/downloading-files-from-a-repository>
- raw.githubusercontent.com:
  <https://docs.github.com/en/repositories/working-with-files/using-files/viewing-and-understanding-files>
- jsDelivr GitHub mirror:
  <https://www.jsdelivr.com/github/>
- x-bash/eget (reference impl of the strategy above):
  <https://github.com/x-bash/eget>

**Verification status**: verified 2024-11 against the above.