---
name: 1-github
description: "GitHub rate limits — quick reference (5 limits + 5 download surfaces). Primary REST 5000/hr (per token / per installation), GraphQL 5000 points/hr cost-based, Search 30/min, Actions 1000/hr/repo, Secondary heuristic. Download surfaces: Releases API counts; HTML scrape counts (UI limit); archive / raw / CDN do NOT count."
type: reference


# Core Content

core_features:

  - "**5 rate limits** at a glance: Primary REST (5000/hr token / 60/hr IP), GraphQL (5000 points/hr cost-based), Search (30/min), Actions (1000/hr/repo), Secondary (heuristic)"
  - "**5 download surfaces** with clear \"counts vs API quota\" column: only Releases API counts"
  - "**Per-token vs per-installation** unit mechanic — PAT/OAuth/user-to-server are per-token; GitHub Apps are per-installation"
  - "**Search / Actions / GraphQL / REST core are separate buckets** — `X-RateLimit-Resource` header tells you which"
  - "**`X-RateLimit-Reset` is UNIX epoch, not relative seconds** (footgun)"
  - "**`Retry-After` is integer seconds** (only on 429)"
  - "**Primary and Secondary 429 look identical** — can't tell from response which bucket fired"
  - HTML scraping has undocumented UI limit (~hundreds/hr/IP) — separate from API bucket but exists

# Key Information

highlights:

  - "**Core strategy**: 1 API call to list releases + CDN to pull assets = 1 API call against budget, unlimited downloads"
  - "`archive` (codeload.github.com), `raw` (raw.githubusercontent.com), and CDN mirrors (jsDelivr, Statically, Gcore) are Fastly-CDN-backed and exempt from GitHub API quota"
  - Per-token = independent budgets (CI isolation); per-installation = aggregate but separate budgets (multi-installation scaling)
  - GraphQL cost = 1-10 points based on highest-cost field (connection and aggregation fields cost more)
  - Avoid bursts (parallel ≤ 5-10) — secondary fires on bursts even with primary headroom
  - "Use conditional requests (`If-None-Match` / `If-Modified-Since`) — 304 doesn't consume quota"

# Use Cases

use_cases:

  - Implementing a GitHub API client with per-token budget tracking
  - Distinguishing primary vs secondary rate-limit triggers (you can't, from the response)
  - Building a CI workflow that respects GitHub App installation-scoped quotas
  - Avoiding secondary rate limits via conditional requests and burst avoidance
  - "**Bypassing GitHub API rate limit** by using raw.githubusercontent.com / codeload.github.com / CDN mirrors instead of the API"
  - Fetching entire repos at scale via codeload.github.com (CDN-backed, no API quota)
  - "**Trimming a repo download** via gitattributes + sparse-checkout for huge repos with LFS / submodules"
  - "**Avoiding download entirely** via cloud dev environments (Codespaces, Gitpod, Colab)"
  - "**Reference implementation**: `x eget` (`x-cmd/x-cmd/mod/eget/lib/api`) — see FAQ `eget-comprehensive-considerations`"

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