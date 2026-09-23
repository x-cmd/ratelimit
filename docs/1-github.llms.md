---
name: 1-github
description: GitHub rate limits (REST 5000/hr auth + 60/hr unauth, GraphQL 5000 pts/hr, Search 30/min, Actions 1000/hr/repo, secondary heuristic) — main article is rate-limit focused. FAQ carries the 5 download surfaces (Releases API / HTML / archive / raw / CDN), x eget comprehensive considerations (the actual code in x-cmd/x-cmd/mod/eget/lib/api), jsDelivr's semver / combine / npm capabilities, gitattributes + sparse-checkout, and cloud dev environments (Colab / Codespaces / Gitpod).
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

# Key Information

highlights:

  - Quota is **per token** for PAT/OAuth/user-to-server, **per installation** for GitHub Apps
  - GraphQL cost: highest-cost field determines cost — connection and aggregation fields cost more
  - `X-RateLimit-Reset` is **UNIX epoch timestamp**, NOT seconds-until-reset (common footgun)
  - `X-RateLimit-Resource` distinguishes which bucket (core / search / graphql / etc.)
  - Secondary rate limit has no published threshold; triggers on abuse-like patterns
  - 429 + Retry-After on both primary AND secondary — same response, different trigger conditions
  - **Article structure**: rate limit is the main axis; download strategies and `x eget` comprehensive considerations live in the FAQ (rendered as page body, not sidebar)

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
  eget-source: https://github.com/x-cmd/x-cmd/tree/X/mod/eget
  eget-api: https://github.com/x-cmd/x-cmd/tree/X/mod/eget/lib/api