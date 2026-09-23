---
name: 6-vercel
description: "Vercel rate limits — function execution quotas (Hobby 100 GB-hr/mo, Pro 1000 GB-hr/mo), Edge Function invocations (500k/mo Hobby, 5M/mo Pro), REST API default 1 RPS, RFC 9745 lowercase ratelimit-* headers, distinguishing 402 (plan-cap) vs 500 (code-error) vs 504 (timeout)."
type: reference


# Core Content

core_features:

- Serverless Function executions: Hobby 100 GB-hr/mo, Pro 1000 GB-hr/mo, Enterprise custom
- Edge Function invocations: Hobby 500k/mo, Pro 5M/mo
- Bandwidth: Hobby 100 GB/mo, Pro 1 TB/mo
- Build minutes: Hobby 100/mo, Pro 400/mo
- Deployments: Hobby 100/day, Pro 3000/day
- REST API default: 1 req/sec per token

## Key Information

highlights:

- GB-hours formula: (memory GB) × (time hours) × (invocations) → consumption
- "Edge Functions and Serverless Functions have **separate quotas**"
- HTTP status codes map to failure modes: 402 = plan-cap exceeded, 500 = code error, 504 = timeout
- "Headers use lowercase `ratelimit-*` (RFC 9745 compliant) — closest to IETF draft"
- Preview deployments unlimited; production deployments capped
- REST API default 1 RPS is the same across plans; differentiation is on product quotas

## Use Cases

use_cases:

- Capacity planning: choose memory and timeout to fit budget
- Distinguishing plan-cap-exceeded (402) from code error (500) in observability
- Auditing whether workload should use Edge Functions vs Serverless Functions
- Designing client libraries that handle RFC 9745 lowercase headers

## Related Resources

official:
  docs: <https://vercel.com/docs/concepts/limits/overview>
  api: <https://vercel.com/docs/rest-api>
related:
  pricing: <https://vercel.com/pricing>
  rfc_9745: <https://datatracker.ietf.org/doc/rfc9745/>