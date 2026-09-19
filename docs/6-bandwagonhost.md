---
x-title: BandwagonHost rate limits — VPS port, bandwidth, and connection caps
x-desc: "BandwagonHost's VPS-level rate-limit equivalents: outbound port 25 (SMTP) blocked by default, monthly bandwidth caps, network port speed, soft connection concurrency limits, and the absence of a public API rate-limit table."
x-sidebar: BandwagonHost rate limits
x-keywords: bandwagonhost, vps, port 25, smtp bandwidth, connection limit, it7 networks
x-json-ld:
  '@context': https://schema.org
  '@graph':
    - '@type': TechArticle
      headline: 'BandwagonHost rate limits'
      inLanguage: 'en'
      about: 'VPS-level rate-limit equivalents for BandwagonHost'
---

# BandwagonHost rate limits — VPS port, bandwidth, and connection caps

BandwagonHost (now part of IT7 Networks) is a **VPS
hosting provider**, not an API service. The "rate limit"
framing that works for Cloudflare / GitHub / Vercel doesn't
perfectly apply here — instead, there are connection-level
and outbound-traffic constraints that *do* feel like rate
limits if your workload pushes them.

## Outbound port 25 (SMTP) — the headline constraint

| Surface | Limit | Notes |
| --- | --- | --- |
| Outbound TCP/25 (SMTP) | **blocked by default** | All VPS plans, per BandwagonHost TOS |

SMTP egress is the single most common reason people run into
"the mail server can't send email" on BandwagonHost VPS
instances. The block exists to keep the IP space from being
abused as a spam source.

**Workarounds:**

1. **Request unblocking through support.** Justify the use
   case (transactional email for a known application, not
   bulk mail). BandwagonHost reviews and unblocks on a
   per-customer basis. This is the cleanest path.

2. **Use a transactional email provider.** SendGrid,
   Mailgun, Postmark, Amazon SES — all provide SMTP
   endpoints that your VPS connects to on port 587 (or
   465 with TLS), which is **not** blocked. The provider
   then handles the actual delivery to recipients.

3. **Use an API-based provider.** Mailgun / Postmark /
   SendGrid / SES all have HTTP APIs that don't use SMTP
   at all. Your VPS makes an HTTPS call, the provider
   delivers the email.

If you really need direct outbound SMTP, **request
unblocking** — trying to work around the block via custom
ports or SMTP-relay-via-Gmail is fragile and TOS-fragile.

## Bandwidth caps

| Plan tier | Monthly outbound traffic | Network port |
| --- | --- | --- |
| Entry-level VPS (1 GB RAM, 20 GB SSD) | 1 TB / month | 1 Gbps |
| Mid-tier VPS (2 GB RAM, 40 GB SSD) | 2 TB / month | 1 Gbps |
| Specialty / larger plans | varies | 10 Gbps (some) |

These are **soft caps** — BandwagonHost will notify you if
you approach the limit and may charge for overage on
larger plans. Hard throttling isn't typical.

If you serve significant traffic from a BandwagonHost VPS
(public-facing service, large file hosting), monitor the
monthly transfer and plan around the cap.

## Concurrent connections

| Surface | Limit | Notes |
| --- | --- | --- |
| Concurrent TCP connections per VPS | varies, soft cap | Plan-dependent; no published hard number |

BandwagonHost doesn't publish a per-plan connection cap.
In practice, the bottleneck is usually:

- The VPS's RAM (each connection consumes some kernel
  state).
- The shared port speed (1 Gbps = ~10k concurrent
  requests/sec at HTTP/1.1 keepalive).

If your workload hits a connection bottleneck, **scale up
the VPS plan** rather than expecting a specific cap to be
raised.

## Control panel API

BandwagonHost's client area (where you manage VPS
instances) has its own internal API rate limits, but they
are **undocumented** and **discouraged for automation**.
Their TOS explicitly notes that automated access to the
client area is reviewed manually.

For programmatic management of BandwagonHost VPS
instances, **use the SolusVM API** (their control panel
backend) if available on your plan, or **use SSH and
configuration management tools** (Ansible, SaltStack,
plain bash) to do the work locally on the VPS itself.

## What BandwagonHost rate-limit story is NOT about

This article does not cover:

- DDoS protection (BandwagonHost's TOS handles abuse
  reports on a per-case basis).
- Inbound traffic (no cap; this is normal for a hosting
  provider).
- CPU / RAM limits (those are VPS-plan specs, not
  rate limits).
- Storage I/O limits (plan-dependent).

These are different concerns; see BandwagonHost's TOS
and plan comparison for details.

## Sources

- Terms of service (port 25 block, bandwidth caps, TOS
  enforcement):
  <https://bandwagonhost.com/terms.php>
- Plans and pricing (per-tier bandwidth caps, port speed):
  <https://bandwagonhost.com/cart.php>
- Network and datacenter info:
  <https://bandwagonhost.com/>
- SolusVM API (control panel backend) — third-party docs:
  <https://docs.solusvm.com/>
- IT7 Networks (parent company):
  <https://www.it7.net/>

**Verification status**: pending CI scraper. Port 25 block
and bandwidth caps are TOS-derived; per-plan numbers must
be cross-checked against the live cart page.
