# Admedi — Vision

**Version:** 1.7
**Repo:** `creational-ai/admedi`
**Display name:** Admedi
**CLI command:** `admedi`
**Python import:** `import admedi`
**Plugin name:** Admedi

## Vision

A unified, open-source Python toolkit and MCP server that lets mobile game studios manage ad mediation configurations from a single config-driven interface. Define your country tiers, waterfall priorities, and floor prices once in a YAML file, then sync across every app, platform, and ad format in your portfolio — via CLI, AI agent (MCP), or Creational.ai's hosted SaaS. The first ad mediation infrastructure-as-code tool, built with an OOP adapter pattern so adding a new mediation platform is just implementing an interface. LevelPlay is the first adapter; MAX and AdMob are future targets, but the abstraction is baked in from day one so adding them is just implementing an interface — not a rewrite.

## Problem Statement

Mobile game studios running ad mediation across multiple apps and platforms face a compounding manual configuration problem. A studio with 6 games on 3 platforms (Android, iOS, Amazon) across 2 ad formats has 36+ configuration surfaces to manage. Every time they want to update country tiers, adjust floor prices, or add a new ad network instance, they're clicking through dashboard UIs repeatedly — the same changes, applied one-by-one.

This problem gets worse as studios grow: more apps, more platforms, more ad formats, more networks. And it multiplies across mediators — many studios run LevelPlay for some titles and MAX or AdMob for others. There is no unified management layer. The existing ironSource Python library (abandoned since Dec 2022) covers only LevelPlay and only as a library — no CLI, no config templating, no multi-mediator abstraction.

**Who this is for:**
- Indie and mid-size mobile game studios running 3+ apps on ad-supported models
- Ad monetization managers who waste hours per week on repetitive dashboard configuration
- Studios running multiple mediators across their portfolio who want a single source of truth

**Why now:**
- ironSource/LevelPlay Python lib abandoned for 3+ years — clear maintenance vacuum
- AppLovin MAX and AdMob Mediation APIs are now mature enough for automation
- MCP protocol enables a new distribution channel (AI-agent-native tooling) that no competitor has touched
- The industry is consolidating around LevelPlay and MAX — two platforms cover the vast majority of the market

## Core Value Proposition

**For the open-source community:** The first config-as-code tool for ad mediation. Define tiers in YAML, version control them, diff before applying, sync across your entire portfolio. Replace 36 dashboard clicks with one command.

**For MCP/AI users:** Expose ad mediation management as MCP tools so AI agents can query performance, suggest tier adjustments, and apply configurations — bringing ad ops into the AI workflow era.

**For Creational.ai SaaS customers:** Managed hosting of the same tool with multi-tenant auth, scheduled syncs, monitoring dashboards, and guaranteed maintenance — for studios that don't want to self-host.

**For Cowork plugin users:** One-click install in Claude desktop. Connect credentials, then manage your entire ad mediation config conversationally — "sync my tier 2 countries across all apps," "show me which apps have mismatched configs," "pull last week's eCPM by country." Non-technical monetization managers get AI-powered ad ops without touching code or a terminal.

**Unique differentiator:** Adapter-based OOP architecture means the tool isn't locked to one mediator. LevelPlay first (because that's our itch), MAX second, AdMob third. The abstraction layer means adding a new mediator is implementing an interface, not rebuilding the tool. This is what Terraform did for cloud infra — we're doing it for ad mediation config.

## Product Evolution (Longer Play)

The config-as-code layer is the wedge. The full flywheel has three layers:

**Layer 1 — Config Management (MVP):** Pull all mediation configs and performance data into one place. Sync tiers, audit drift, manage instances across the portfolio. This is the infrastructure-as-code play — the Terraform analogy. Ship this first, dogfood on Mochibits' Shelf Sort portfolio.

**Layer 2 — AI-Powered Intelligence:** Once all the data lives in the system — eCPMs by country, fill rates by network, revenue by tier, config history — Claude can reason about it. The AI looks at Tier 3 performance, sees Taiwan outperforming by 40%, and suggests promoting it to Tier 2. Data-backed recommendations generated from your own portfolio data, not a black box.

**Layer 3 — Automated A/B Testing:** Close the loop. The AI proposes a thesis ("South Korea should move to Tier 2 based on $10.50 avg eCPM over 30 days"), you approve, Admedi applies the change to a subset of apps as a test while holding others as control, then measures the result over a defined period. Propose → Approve → Execute → Measure → Learn.

This is the moat Bidlogic can't replicate — they own the data and hide the reasoning. With Admedi, studios own their data, see the thesis, approve the changes, and build institutional knowledge. Over time, anonymized benchmarks across the user base create a data network effect: the more studios use it, the smarter recommendations get for everyone.

**Important:** Layers 2 and 3 are the longer play. MVP focus is Layer 1 only — get the config engine working for Mochibits first, open-source it, then build intelligence on top.

## Success Metrics

**MVP (Mochibits internal — Layer 1 only):**

| Metric | Target | How Measured |
|--------|--------|--------------|
| Shelf Sort portfolio fully managed | All 6 apps × 3 platforms synced via CLI | Manual verification — tier configs match across all 18 surfaces |
| Config sync time | < 2 minutes for full portfolio sync (vs ~45 min manual) | CLI timing output |
| LevelPlay API coverage | 100% of management endpoints (Groups v4, Instances, Placements, Reporting) | Endpoint test coverage |
| Zero config drift | Audit command catches any mismatch between template and live config | Audit report output |

**Post-MVP (open-source + SaaS):**

| Metric | Target | How Measured |
|--------|--------|--------------|
| Open-source GitHub stars | 50 within first 3 months | GitHub metrics |
| MCP tool adoption | 5+ external users connecting via MCP within 6 months | MCP server connection logs |
| Second mediator adapter (MAX) | Functional GET + PUT for groups/waterfall | Integration tests |
| SaaS waitlist | 20 studios within 6 months of open-source launch | Waitlist signups on Creational.ai |

## Non-Goals

- **Not a mediation SDK or client-side library.** This is a server-side management and configuration tool. It does not replace the LevelPlay/MAX/AdMob SDKs in your app.
- **Not a real-time bidding optimizer.** Tools like Bidlogic do ML-driven real-time waterfall optimization. We're focused on configuration management (infrastructure-as-code), not algorithmic optimization.
- **Not a reporting dashboard.** We expose reporting APIs for data pull (useful for AI agents and scripts), but we're not building a competing analytics UI. The mediator dashboards are fine for visualization.
- **Not a creative/ad content tool.** We manage where and how ads are served (mediation config), not what ads are shown.
- **Not multi-tenant SaaS at MVP.** MVP is single-user CLI/MCP for our own portfolio. SaaS comes after validation.

## Risks & Mitigations

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| LevelPlay API deprecation or breaking changes (Unity is known for API churn) | M | H | Fork from existing lib, pin to known-good versions, add version detection. The v2 → v4 migration pattern is already documented — we support both. |
| MAX API access may be restricted or require partnership | M | H | Research MAX API availability before committing to adapter. If gated, deprioritize and focus on LevelPlay + AdMob. |
| Low open-source adoption — niche audience | M | M | The tool solves our own problem first. Open-source is distribution, not the business model. SaaS is the revenue path. |
| Mediator API rate limits constrain bulk operations | L | M | Already documented: LevelPlay = 4K/30min, more than enough for 18 surfaces. Implement exponential backoff and batch where APIs support it. |
| Scope creep into optimization/ML territory | M | M | Hard boundary in non-goals. Config management only. If demand exists for optimization, that's a separate product. |
| Security risk of storing API credentials | M | H | .env-based config with clear documentation. SaaS version uses encrypted credential storage. Never log or transmit credentials. |

## Resolved Decisions

**Multi-mediator scope:** Build with abstraction from day one (adapter interface pattern), but only implement the LevelPlay adapter for MVP. MAX and AdMob adapters come later — when we're ready, adding a mediator should be as simple as implementing the interface and having the API docs. The burden is on each ad network to provide API access.

**Repo strategy:** Brand new repo on Creational.ai GitHub org. Clone the useful patterns from `ironSource/mobile-api-lib-python` (auth flow, HTTP client, data models) but rebuild clean — no fork baggage.

**MCP tool naming:** Generic tool names only. No mediator prefixes in tool names. The mediator connection is a top-level configuration concern (`.env` / config file), not a per-tool concern. Tools are named by function: `get_groups`, `sync_tiers`, `get_reporting` — not `levelplay_get_groups`. This keeps the MCP interface stable as new adapters are added.

**Licensing model:** Apache-2.0 for the open-source core (matching the original ironSource lib and the Supabase model). Enterprise/SaaS features (multi-tenant auth, scheduled syncs, monitoring) live in a separate `/ee` directory with a commercial license. This is the proven open-core pattern used by Supabase (Apache-2.0 core + commercial `/ee`), Cal.com, and PostHog. It protects the SaaS offering while keeping the community tool fully open.

The pattern:
- `/` — Apache-2.0 (CLI, MCP server, adapters, config engine)
- `/ee` — Commercial license (multi-tenant, SaaS hosting, enterprise features)

**Infrastructure & deployment:** SaaS runs on Creational.ai's existing AWS stack — App Runner + RDS (Postgres), same pattern as Video Professor and Mission Control. No new infra to invent.

**Storage adapter pattern:** Two adapter boundaries in the architecture, not one:
- **Mediation adapters** — LevelPlay, (future: MAX, AdMob). Abstracts the API layer per platform.
- **Storage adapters** — Abstracts persistence. Open-source users get local file (YAML/JSON) or SQLite for zero-dependency self-hosting. SaaS connects to the existing RDS Postgres on Creational.ai's AWS infrastructure. Same interface, swap via config.

This means the open-source experience is truly zero-dependency: `pip install`, `.env` for mediation credentials, local file storage by default. SaaS customers get managed Postgres with full audit history, config versioning, multi-tenant isolation, and scheduled syncs — all backed by the same RDS that already powers the Creational.ai ecosystem.

## Distribution Channels

The same core engine packaged four ways:

| Channel | Audience | Install | Interface |
|---------|----------|---------|-----------|
| **Python library** | Developers who want to script/automate | `pip install git+https://github.com/creational-ai/admedi` | `import admedi` |
| **CLI** | Terminal-driven ad ops / DevOps | Same install, then `admedi sync-tiers` | Command line |
| **MCP server** | AI agent users (Claude Code, Cursor, etc.) | `.env` config + install from GitHub | AI conversation via tools |
| **Cowork / Claude Desktop plugin** | Non-technical monetization managers | One-click install in Claude desktop | Conversational AI with guided workflows |

### Plugin Skills (ship with the Cowork plugin)

| Skill | Trigger | What It Does |
|-------|---------|--------------|
| `setup` | "connect my LevelPlay account", "set up ad mediation" | Guided onboarding: credential entry, app discovery, initial config pull |
| `sync-tiers` | "sync tiers across all apps", "apply tier template" | Loads tier template from config, shows diff of what will change per app, confirms, applies |
| `audit-config` | "check my configs", "find mismatches" | Pulls configs from all apps, compares against template, flags inconsistencies (e.g., App X is missing South Korea in Tier 2) |
| `revenue-check` | "how are my tiers performing", "eCPM by country" | Pulls reporting data, surfaces underperforming tiers/countries, suggests tier adjustments based on actual eCPM data |
| `manage-instances` | "add Pangle to all US waterfalls", "disable InMobi" | Bulk instance management across apps — add, remove, enable/disable ad network instances |

Each skill is a conversational workflow that wraps the MCP tools underneath. The skill handles the UX — showing previews, asking for confirmation before destructive actions, explaining what changed. The MCP tools handle the API calls.

## Package Name — RESOLVED

**Name:** `Admedi`
**Repo:** `creational-ai/admedi`
**CLI command:** `admedi`
**Import:** `import admedi`

**Why Admedi:** Positions the product as an intelligence layer for ad mediation — not just config sync, but data-informed decision making that integrates with AI (Claude). The name scales naturally: starts as config management, grows into reporting insights, eCPM-driven tier suggestions, and AI-powered audit workflows. "Admedi powered by Claude" and "Admedi MCP" both land naturally. For SaaS positioning, you're selling ad intelligence, not a YAML pusher.

**Distribution:** Install directly from GitHub — no PyPI overhead, no name conflicts.
```
pip install git+https://github.com/creational-ai/admedi
```

## MVP Scope — Mochibits First

The immediate priority is building Admedi as an internal tool for Mochibits' Shelf Sort portfolio (6 apps × 3 platforms = 18 configuration surfaces on LevelPlay). Everything else — open-source launch, SaaS, MAX adapter, AI optimization layers — comes after the tool is working and battle-tested on our own portfolio.

**MVP deliverables:**
- LevelPlay adapter (Groups v4, Instances v3, Placements v1, Reporting v1)
- ConfigEngine (Loader → Differ → Applier)
- YAML tier template format
- CLI commands: `sync-tiers`, `audit`, `revenue`, `manage-instances`
- MCP server with generic tool names
- Local file storage adapter (default)

**Not in MVP:**
- SaaS hosting / multi-tenant
- MAX or AdMob adapters
- Cowork plugin
- AI optimization (Layer 2/3)
- Postgres storage adapter

## Open Questions

- [ ] MAX and AdMob API research — deferred to post-MVP, but needed before committing adapter dev time
- [ ] Validate YAML tier template format against real Shelf Sort config before building the full engine
