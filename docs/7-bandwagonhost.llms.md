---
name: 6-bandwagonhost
description: BandwagonHost (VPS hosting) rate-limit equivalents — outbound port 25 (SMTP) blocked by default, monthly bandwidth caps (1 TB/mo entry-level), 1 Gbps shared port, soft concurrent TCP connection caps, no public API rate-limit table.
type: reference
---

# Core Content

core_features:

- Outbound port 25 (SMTP): **blocked by default** on all VPS plans (anti-spam policy)
- Monthly outbound bandwidth: 1 TB/mo entry-level, scales up; soft cap with notification
- Network port speed: 1 Gbps shared on most plans; 10 Gbps on some specialty plans
- Concurrent TCP connections: soft cap, no published hard number per plan
- No public programmatic API; control panel discouraged for automation

## Key Information

highlights:

- Port 25 block applies to **outbound** SMTP (sending mail from VPS) — inbound to VPS is fine
- Workarounds for SMTP egress: support request unblock (justified use case), SMTP relay via provider port 587, or HTTP API (SendGrid / Mailgun / SES)
- Bandwidth caps are soft — notification, possible overage on larger plans; hard throttling atypical
- Connection caps depend on RAM + shared port speed, not a published hard number
- SolusVM API available per plan for programmatic management; or SSH + config management
- "Rate limit" framing partially applies (port 25, bandwidth, soft connections); per-endpoint API quota framing doesn't

## Use Cases

use_cases:

- Diagnosing "can't send mail from VPS" — most likely port 25 block
- Planning outbound traffic budget for public-facing services on a BandwagonHost VPS
- Choosing between direct SMTP (with unblock request) vs transactional email provider
- Migrating from "I should be able to send mail" assumption to port-587 relay reality

## Related Resources

official:
  tos: <https://bandwagonhost.com/terms.php>
  cart: <https://bandwagonhost.com/cart.php>
related:
  smtp_relays: <https://sendgrid.com>, <https://postmarkapp.com>, <https://aws.amazon.com/ses/>
  solusvm: <https://docs.solusvm.com/>