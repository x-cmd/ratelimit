---
x-title: Vercel rate limits — function executions, edge quotas, REST API
x-desc: Vercel's per-plan function / edge function quotas (100 GB-hr/mo Hobby, 1 TB-hr/mo Pro), REST API default 1 RPS, RFC 9745 lowercase `ratelimit-*` headers, and how to design around the function-execution limits.
x-sidebar: Vercel rate limits
x-keywords: vercel, ratelimit, gb-hr, serverless function, edge function, bandwidth, deployments, rfc 9745
x-json-ld:
  '@context': https://schema.org
  '@graph':
    - '@type': TechArticle
      headline: 'Vercel rate limits'
      inLanguage: 'en'
      about: 'Vercel function, edge, and API rate limits'
---

# Vercel rate limits — function executions, edge quotas, REST API

Vercel has a different shape from API-rate-limit-first
vendors like Cloudflare or GitHub. The primary
constraints are **function execution budget** (memory ×
time × invocations) and **bandwidth**, with the API rate
limit being secondary.

## Plan-by-plan product limits

| Limit | Hobby (free) | Pro | Enterprise |
| --- | --- | --- | --- |
| Serverless function executions | 100 GB-hr / month | 1,000 GB-hr / month | custom |
| Serverless function timeout (default) | 10s | 60s | 900s |
| Edge function invocations | 500,000 / month | 5,000,000 / month | custom |
| Edge function code size | 1 MB | 4 MB | custom |
| Bandwidth (Fast Data Transfer) | 100 GB / month | 1 TB / month | custom |
| Build minutes | 100 / month | 400 / month | custom |
| Deployments (production) | 100 / day | 3,000 / day | custom |
| Preview deployments | unlimited | unlimited | unlimited |

Numbers above are the published plan limits as of
training-data knowledge; verify against
`vercel.com/docs/concepts/limits/overview`.

## Understanding "GB-hours"

Vercel's execution quota is measured in **GB-hours**,
defined as:

```
GB-hours = (memory in GB) × (execution time in hours)
         × (number of invocations)
```

Example: a function configured with 1 GB memory, executed
100,000 times for an average of 1 second each:

```
1 GB × (1 / 3600) hours × 100,000 invocations
= 27.78 GB-hours
```

A Hobby-plan quota of 100 GB-hours would allow ~3.6× this
workload. Pro plan: 36×.

## Vercel REST API

| Surface | Default | Per |
| --- | --- | --- |
| Vercel REST API | 1 req / sec | token |

Most Vercel REST endpoints share this default. Batched
endpoints (`/v13/bulk-deployments` etc.) typically allow
higher. The API itself is documented at
`vercel.com/docs/rest-api`.

## Headers Vercel emits

Vercel's REST API uses the lowercase `RateLimit-*` header
naming, matching the IETF draft (RFC 9745):

```http
ratelimit-limit: 60
ratelimit-remaining: 59
ratelimit-reset: 60
retry-after: 60
```

This is the cleanest header format of any vendor in this
repo — `retry-after` is in seconds (integer), `ratelimit-*`
follows the IETF draft exactly.

## Practical patterns

### Choosing memory and timeout to fit budget

If you're on Hobby (100 GB-hr/mo) and expect 1M
invocations/month:

```
Memory × Time_per_invocation × 1M ≤ 100 GB-hours
```

For 1M invocations and 100 GB-hr:

```
Memory × Time ≤ 3.6 × 10⁻⁴ GB-hour
         = 1.3 second-GB per invocation

So a 256 MB function with 5s timeout = 1.28 GB-second,
which would exhaust budget at ~78k invocations/month.
A 128 MB function with 2s timeout = 0.256 GB-second, good
for ~390k invocations/month.
```

The trade-off: lower memory = lower per-invocation cost but
higher risk of OOM crashes on memory-intensive paths.

### Distinguishing "Hobby-cap-exceeded" vs "code error"

Vercel returns:
- HTTP 402 for plan-cap-exceeded (Hobby's 100 GB-hr).
- HTTP 500 for function code errors.
- HTTP 504 for function timeouts.

Distinguishing these is essential: a code-error
notification should page the on-call; a plan-cap-exceeded
notification should notify the billing / growth team.

### Edge Functions vs Serverless Functions

These are **separate quotas**. A project on Hobby has:
- 100 GB-hr/mo for Serverless Functions (Node.js / Python
  / Go / etc.)
- 500,000 invocations/mo for Edge Functions (V8 isolates)

Edge Functions are cheaper per-invocation but have a hard
1MB code-size limit and a different runtime model. Use
Edge Functions for request-time middleware (auth,
routing, A/B test assignment) and Serverless Functions
for compute-heavy paths (image processing, PDF generation).

## What this article does NOT cover

- Vercel's DDoS protection (separate layer).
- Build image caching limits.
- Log retention quotas.

These are different topics; see Vercel docs.

## Sources

- Limits overview (function / Edge / bandwidth / build / deploy):
  <https://vercel.com/docs/concepts/limits/overview>
- REST API:
  <https://vercel.com/docs/rest-api>
- Pricing:
  <https://vercel.com/pricing>
- Vercel Edge Functions runtime:
  <https://vercel.com/docs/functions/edge-functions>
- Serverless Functions:
  <https://vercel.com/docs/functions/serverless-functions>
