# Contributing

This repo is the producer side of the rate-limit reference.
The consumer side is the [`x ratelimit`](https://x-cmd.com/mod/ratelimit)
shell module (forthcoming) and the `x-cmd.com/ratelimit` site.

External PRs are welcome for:

- **Data corrections** — wrong number, outdated plan tier,
  missing endpoint. Open a PR; the team reviews within a few
  days.
- **New vendor files** — copy an existing YAML as a
  template, fill in the fields, open a PR.
- **Article edits** — corrections, additional examples,
  vendor-specific deep dives.

## Repository layout

```text
.
├── LICENSE                  ← Apache 2.0
├── LICENSE-data             ← CC-BY-4.0 (data files)
├── README.md / README.cn.md
├── SKILL.md                ← AI-agent recipe
├── CONTRIBUTING.md         ← this file
├── RATELIMIT-RESEARCH.md   ← working notes, verification status
├── data/                   ← per-vendor YAML (source of truth)
│   ├── cloudflare.yaml
│   ├── aliyun.yaml
│   ├── tencent.yaml
│   ├── github.yaml
│   ├── vercel.yaml
│   └── bandwagonhost.yaml
├── docs/                   ← long-form articles (served at x-cmd.com/ratelimit)
│   ├── 0-ratelimit-overview.{en.md,cn.md,llms.md,faq.yml}
│   ├── 1-cloudflare.{en.md,cn.md,llms.md,faq.yml}
│   ├── 2-aliyun.{en.md,cn.md,llms.md,faq.yml}
│   ├── 3-tencent.{en.md,cn.md,llms.md,faq.yml}
│   ├── 4-github.{en.md,cn.md,llms.md,faq.yml}
│   ├── 5-vercel.{en.md,cn.md,llms.md,faq.yml}
│   ├── 6-bandwagonhost.{en.md,cn.md,llms.md,faq.yml}
│   └── 7-rate-limit-headers-cheatsheet.{en.md,cn.md,llms.md,faq.yml}
└── .github/
    └── workflows/
        └── (scrape.yml will go here)
```

## Sign-off (DCO)

This repo uses [DCO](https://developercertificate.org/),
not a CLA. Every commit you push to a branch and every PR
you open here should include a `Signed-off-by:` trailer:

```text
Signed-off-by: Your Name <you@example.com>
```

The DCO certifies that you wrote the contribution or have
the right to submit it under the project's license. Git
makes this easy:

```sh
git commit -s -m "data: fix Cloudflare Pro plan request cap"
```

The `-s` flag adds the `Signed-off-by:` trailer
automatically.

The CI will reject commits without the trailer. We do NOT
require a separate Contributor License Agreement (CLA) — DCO
is sufficient for this size of project. If you want to grant
additional rights (e.g., transfer copyright, dual-license),
that's a separate conversation; open an issue.

## Per-vendor YAML schema

Each `data/<vendor>.yaml` has this structure:

```yaml
vendor: <display name>
slug: <kebab-case>            # matches the filename
api_base: <base URL>          # or null if N/A
docs_source: <official doc URL>
verified: true | false         # team has confirmed against official source
last_verified: YYYY-MM-DD | null
pricing_notes: ...             # free-text pricing context
plans:
  - name: <plan display>
    api_rate_limit: {...}      # the headline API quota
    product_limits: [...]      # secondary caps (Workers, KV, etc.)
    rate_limit_headers: {...}  # headers this plan emits
header_conventions: ...        # cross-vendor header summary
```

Fields marked `verified: false` are pending CI scraper
verification. PRs that change `verified: false → true` after
the team cross-checks are very welcome.

## Adding a new vendor

1. Copy `data/cloudflare.yaml` (or another existing file)
   as a template.
2. Fill in the schema above with values from the vendor's
   official rate-limit documentation.
3. Add the vendor slug to the README's "Vendors covered"
   table.
4. Add a per-vendor article under `docs/` (English + Chinese
   + LLM summary + FAQ YAML).
5. Open a PR.

## Three concrete consequences

1. **Bug reports / wrong numbers** — issue, not PR.
2. **New vendor files** — issue or PR; the team reviews.
3. **Anything touching `LICENSE`, `LICENSE-data`,
   `.github/workflows/`, or `data/` schema** — issue first;
   the team reviews the structural change before code lands.

## What to read next

- [`RATELIMIT-RESEARCH.md`](./RATELIMIT-RESEARCH.md) — current
  verification status per vendor.
- [`docs/0-ratelimit-overview.en.md`](./docs/0-ratelimit-overview.en.md) —
  how to use this repo end-to-end.
- [`SKILL.md`](./SKILL.md) — AI-agent recipes.

## License of contributions

- Code, prose, scripts: Apache 2.0.
- Data tables under `data/`: CC-BY-4.0.

By signing off a commit (DCO `Signed-off-by:`), you agree
the contribution is licensed under the matching license for
the path it lands on.