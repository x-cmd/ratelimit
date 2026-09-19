---
name: 4-github
description: GitHub rate limits — primary REST (5000/hr authenticated, 60/hr unauthenticated), GraphQL (5000 points/hr), Search (30/min), Actions (1000/hr/repo), secondary rate limit (heuristic abuse detection), X-RateLimit-* headers with epoch timestamp semantics.
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

## Key Information

highlights:

- Quota is **per token** for PAT/OAuth/user-to-server, **per installation** for GitHub Apps
- GraphQL cost: highest-cost field determines cost — connection and aggregation fields cost more
- `X-RateLimit-Reset` is **UNIX epoch timestamp**, NOT seconds-until-reset (common footgun)
- `X-RateLimit-Resource` distinguishes which bucket (core / search / graphql / etc.)
- Secondary rate limit has no published threshold; triggers on abuse-like patterns
- 429 + Retry-After on both primary AND secondary — same response, different trigger conditions

## Use Cases

use_cases:

- Implementing a GitHub API client with proper per-token budget tracking
- Distinguishing primary vs secondary rate limit triggers from response patterns
- Building a CI workflow that respects GitHub App installation-scoped quotas
- Avoiding secondary rate limits via conditional requests (If-None-Match)

## Related Resources

official:
  rest: <https://docs.github.com/en/rest/using-the-rest-api/rate-limits-for-the-rest-api>
  graphql: <https://docs.github.com/en/graphql/overview/resource-limitations>
  secondary: <https://github.blog/developer-skills/github/how-to-prevent-secondary-rate-limit-issues/>
related:
  actions: <https://docs.github.com/en/rest/actions>
  search: <https://docs.github.com/en/rest/search>