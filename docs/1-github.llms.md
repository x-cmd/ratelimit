---
name: 1-github
description: GitHub rate limits (REST 5000/hr auth + 60/hr unauth, GraphQL 5000 pts/hr, Search 30/min, Actions 1000/hr/repo, secondary heuristic) and five download strategies: Releases API, HTML scraping, archive/tarball, raw content (raw.githubusercontent.com), and CDN mirrors (jsDelivr / Statically / gcore). Plus developer angle: gitattributes + sparse-checkout + cloud dev environments (Colab / Codespaces / Gitpod). x-bash/eget reference impl.
type: reference
---

# Core Content

core_features:

  - REST primary: 5000 req/hr per token (PAT, OAuth, GitHub App)
  - REST unauthenticated: 60 req/hr per IP
  - GraphQL: 5000 points/hr (cost-based, 1-10 points per query depending on highest-cost field)
  - Search API: 30 req/min per user (separate from REST primary)
  - Actions API: 1000 req/hr per repo
  - Secondary rate limit: heuristic abuse detection (bursts, concurrent in-flight, repeated identical content)
  - **Five download surfaces**, each with its own rate-limit story:
    - Releases API (`api.github.com`) — counts 5000/hr auth or 60/hr unauth
    - HTML scraping (`github.com/.../releases/...`) — undocumented UI limit ~hundreds/hr per IP, DOM drift
    - Archive tarball (`codeload.github.com/.../tar.gz/refs/...` or `github.com/.../archive/refs/.../tar.gz`) — Fastly CDN, no documented GitHub API limit, ≤100 MB cap
    - Raw content (`raw.githubusercontent.com/...`) — Fastly CDN, no documented GitHub API limit, soft limit ~60-100 req/min per IP
    - CDN mirrors (`cdn.jsdelivr.net/gh/...`, `cdn.statically.io/gh/...`, `gcore.jsdelivr.net/gh/...`) — CDN-level (~50M req/month free tier), eventually consistent
  - Developer angle: gitattributes + sparse-checkout (`git clone --depth=1 --filter=blob:none --sparse`) + cloud dev environments (GitHub Codespaces, Gitpod, Google Colab)

# Key Information

highlights:

  - Quota is **per token** for PAT/OAuth/user-to-server, **per installation** for GitHub Apps
  - GraphQL cost: highest-cost field determines cost — connection and aggregation fields cost more
  - `X-RateLimit-Reset` is **UNIX epoch timestamp**, NOT seconds-until-reset (common footgun)
  - `X-RateLimit-Resource` distinguishes which bucket (core / search / graphql / etc.)
  - Secondary rate limit has no published threshold; triggers on abuse-like patterns
  - 429 + Retry-After on both primary AND secondary — same response, different trigger conditions
  - **Picking the right download surface saves API budget**: `raw.xxx` and `codeload.github.com` are CDN-backed, NOT counted against GitHub API quota
  - **Only the Releases API counts** — the other four download surfaces are CDN-cached and exempt
  - `x eget` (x-bash/eget) implements this exact strategy: API for listing → archive/raw for downloading

# Use Cases

use_cases:

  - Implementing a GitHub API client with proper per-token budget tracking
  - Distinguishing primary vs secondary rate limit triggers from response patterns
  - Building a CI workflow that respects GitHub App installation-scoped quotas
  - Avoiding secondary rate limits via conditional requests (If-None-Match)
  - **Bypassing GitHub API rate limit** by using raw.githubusercontent.com or archive tarballs instead of the API
  - Fetching entire repos at scale via codeload.github.com (CDN-backed, no API quota)
  - **Trimming a repo download** via gitattributes + sparse-checkout for huge repos with LFS / submodules
  - **Avoiding download entirely** via cloud dev environments (Codespaces, Gitpod, Colab)

# Related Resources

official:
  rest: https://docs.github.com/en/rest/using-the-rest-api/rate-limits-for-the-rest-api
  graphql: https://docs.github.com/en/graphql/overview/resource-limitations
  secondary: https://github.blog/developer-skills/github/how-to-prevent-secondary-rate-limit-issues/
  raw: https://docs.github.com/en/repositories/working-with-files/using-files/viewing-and-understanding-files
  codeload: https://docs.github.com/en/repositories/working-with-files/using-files/downloading-files-from-a-repository
related:
  actions: https://docs.github.com/en/rest/actions
  search: https://docs.github.com/en/rest/search
  jsdelivr: https://www.jsdelivr.com/github/
  eget: https://github.com/x-bash/eget
  static: https://cdn.statically.io