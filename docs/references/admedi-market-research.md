# Admedi — Market Research

**Version:** 1.0
**Date**: February 26, 2026

## Context Source

**Primary source**: Documents produced during design sessions

- `admedi-vision.md` v1.6 — Product vision, problem statement, value proposition, distribution channels
- `admedi-architecture.md` v1.0 — Technical architecture, tech stack, data model, integration points
- `levelplay-api-reference.md` v1.1 — API surface documentation

| Field | Value |
|-------|-------|
| `objective` | Config-as-code tool for ad mediation management — CLI, MCP server, Cowork plugin, SaaS |
| `target_market` | Indie and mid-size mobile game studios running 3+ ad-supported apps |
| `revenue_model` | Open-core: Apache-2.0 CLI/MCP + commercial SaaS with managed hosting |
| `monthly_cost` | ~$15/mo baseline (App Runner $10 + RDS $0 + Secrets $0.80) — same Creational.ai AWS stack |
| `projected_mrr` | Not yet defined — this report establishes pricing recommendations |
| `architecture_summary` | Dual adapter pattern (Mediation + Storage), ConfigEngine (Loader → Differ → Applier), 4 interfaces |

---

## Executive Summary

**Recommendation**: GO
**Confidence**: High

Admedi occupies a genuine whitespace in the mobile ad mediation ecosystem. There are zero config-as-code tools for ad mediation today. The existing competitors — Bidlogic, Superscale, and GameBiz Consulting — all operate as managed services or consultancies that charge revenue share on ad uplift. None offer a self-serve developer tool, and none expose automation via CLI, API library, or AI agent (MCP). The ironSource Python library that once covered some of this surface has been abandoned since December 2022, leaving a maintenance vacuum with no replacement.

The ad mediation platform market itself ($2.23B in 2024, 13.7% CAGR) is healthy and growing, but Admedi's real market is the tooling layer on top — closer to a developer infrastructure play (Terraform for ad mediation) than a mediation platform. The SaaS opportunity is strongest as a flat subscription model priced per managed app, targeting the $49–149/mo range that sits well below the cost of hiring even a part-time ad monetization specialist ($148K/yr average salary). The open-source core drives distribution and credibility, while the SaaS captures studios that want managed persistence, scheduled syncs, and zero-ops.

Key success factors: ship the open-source CLI fast, dogfood on the Shelf Sort portfolio (6 apps × 3 platforms = 18 surfaces), and use the working product as the marketing asset in r/gamedev, PG Connects, and GDC circles.

---

## Market Landscape

### Market Category

Admedi sits at the intersection of two categories: **mobile ad mediation platforms** (LevelPlay, MAX, AdMob) and **developer infrastructure tooling** (Terraform, Pulumi, config management). It is not a mediation platform itself — it is a management and configuration layer that sits on top of existing mediators. The closest analogy is Terraform: Terraform doesn't replace AWS, it manages AWS configurations as code. Admedi doesn't replace LevelPlay, it manages LevelPlay configurations as code.

### Market Size

| Segment | Size | Source |
|---------|------|--------|
| **TAM** (Mobile Ad Mediation Platform Market) | $2.23B (2024) → $6.87B by 2033 | [Growth Market Reports](https://growthmarketreports.com/report/mobile-ad-mediation-platform-market) |
| **SAM** (Ad Mediation for Games specifically) | $1.42B (2024) → $4.22B by 2033 | [Growth Market Reports](https://growthmarketreports.com/report/ad-mediation-platforms-for-games-market) |
| **SOM** (Studios who would pay for config management tooling) | ~2,000–5,000 studios × $49–149/mo = $1.2M–$8.9M ARR addressable | Reasoning below |

**SOM Reasoning**: Google Play hosts ~2.87M active publishers. The game category represents roughly 15–20% of publishers. Of those, studios running 3+ ad-supported apps with enough scale to justify mediation tooling is a small subset — estimated at 5,000–10,000 globally. Of those reachable by our channels (open-source discovery, PG Connects, r/gamedev, MCP ecosystem), perhaps 2,000–5,000 in the first 2 years. At 50–200 paying SaaS customers in the first year, that's $30K–$360K ARR, scaling toward $1M+ as the product matures.

### Market Trends

**Growth**: Growing — 13.7% CAGR for ad mediation platforms, 7.9–8.1% CAGR for in-app advertising broadly.

**Tailwinds** (forces helping us):

- **Abandoned tooling vacuum**: The ironSource Python library (the only open-source management tool) has been abandoned since Dec 2022. No replacement exists. Studios have zero automation options besides building internal scripts.
- **API maturity**: Both LevelPlay (Groups API v4, full REST surface) and AppLovin MAX (new Ad Unit Management API launched 2024) now have mature programmatic APIs. The infrastructure to build on exists.
- **MCP/AI-agent adoption**: The MCP protocol is gaining traction as the standard for AI-tool integration. Being the first ad mediation MCP tool is a timing advantage that compounds — early MCP tools become the defaults.
- **Industry consolidation**: The market is consolidating around LevelPlay and MAX. Two adapters cover the vast majority of the market, making multi-mediator support tractable.
- **Waterfall management pain**: Industry sources consistently describe waterfall management as tedious, manual, and error-prone. The shift toward hybrid bidding + waterfall models adds complexity, not removes it.

**Headwinds** (forces against us):

- **Bidding reduces waterfall complexity**: As bidding adoption increases, the manual waterfall configuration that Admedi automates becomes less critical for some studios. However, hybrid models (bidding + waterfall) still require configuration, and country tiers / floor prices still need management.
- **Platform lock-in**: LevelPlay and MAX could build native config management features into their dashboards, reducing the need for external tools. However, neither has done so in 3+ years, and cross-mediator tooling would remain valuable.
- **Niche audience**: The audience for ad mediation config tooling is inherently small — only studios with multiple apps and platforms benefit enough to justify adoption.

### Timing Assessment

The timing is strong. Three converging factors create an unusual window: the ironSource lib has been abandoned for 3+ years (vacuum), mediation APIs are newly mature enough for full automation (capability), and the MCP protocol creates a novel distribution channel (reach). The risk of being "too late" is low because nobody has built this. The risk of being "too early" is also low because the APIs already exist. The main timing risk is that LevelPlay or MAX could build native bulk management features, but that hasn't happened in the time since the APIs launched.

---

## Competitive Analysis

### Landscape Map

| Competitor | Category | Positioning | Pricing | Strengths | Weaknesses |
|------------|----------|-------------|---------|-----------|------------|
| Bidlogic | Direct (optimization) | "Ad mediation automation for mobile apps" | Revenue share / custom | ML-driven waterfall optimization, supports LevelPlay + MAX | Black box, no self-serve, not config-as-code, custom pricing |
| Superscale | Direct (managed service) | "Manage your game for share of uplift" | Revenue share, no upfront | Full-service monetization management | Requires handing over control, revenue share model, not a tool |
| GameBiz Consulting | Indirect (consulting) | "Ad monetization specialist services" | Custom consulting fees | Human expertise, proven track record, 7-figure portfolio experience | Not scalable, expensive, no tooling |
| ironSource Python lib | Indirect (abandoned OSS) | Python API wrapper | Free (OSS) | 18 endpoints implemented, OAuth auth, async HTTP | Abandoned since Dec 2022, no CLI, no config engine, no MCP |
| Internal scripts | Indirect (DIY) | Custom per-studio | Engineering time | Perfectly tailored | Unmaintainable, no community, duplicated effort across industry |

### Competitor Deep Dives

#### Bidlogic (Most Relevant)

- **What they do**: Automated waterfall and bidding optimization using ML. Creates and manages eCPM floor prices, waterfall ordering, and A/B tests programmatically.
- **Target customer**: Mid-to-large mobile game studios with significant ad revenue
- **Pricing**: Revenue share model (% of ad revenue uplift), custom pricing per client. First month free if adding optAd360 demand. No published self-serve pricing.
- **Strengths**: Sophisticated optimization algorithms, supports both LevelPlay and MAX, proven revenue uplift claims, handles the hard optimization problem
- **Weaknesses**: Opaque pricing, requires giving up control, not open-source, no CLI or developer tooling, no config-as-code, no audit trail, no AI/MCP integration. Studios must trust Bidlogic's black box.
- **User sentiment**: Limited public reviews. Positioned as premium service, not accessible to indie studios.

#### Superscale

- **What they do**: Full-service mobile game growth management including ad mediation, UA, analytics, and live ops
- **Target customer**: Studios that want to outsource monetization management entirely
- **Pricing**: Revenue share on uplift, no upfront cost
- **Strengths**: Comprehensive service, no upfront risk, expert team
- **Weaknesses**: Requires ceding control, revenue share eats into margins long-term, not a tool — it's outsourcing. No self-serve option.
- **User sentiment**: Positioned for studios that lack in-house monetization expertise.

#### GameBiz Consulting

- **What they do**: Boutique consulting for mobile game ad monetization — strategy, setup, ongoing management
- **Target customer**: Studios that want expert guidance but retain control
- **Pricing**: Custom consulting fees, flexible models
- **Strengths**: Deep expertise, hands-on, 7-figure portfolio track record
- **Weaknesses**: Doesn't scale. Consulting hours are finite. No product or tool to sell.

### Gap Analysis

**Underserved segments**:

- **Self-serve studios**: Studios with in-house ad ops capability (even if it's one person) that want tooling, not a managed service. Bidlogic and Superscale target studios that want to hand over control. There's nothing for studios that want to keep control but reduce the manual work.
- **Indie/small studios**: Studios running 3–10 apps can't justify Bidlogic's custom pricing or Superscale's revenue share. They need a $50–150/mo tool, not a revenue share arrangement.
- **Developer-oriented ops**: No competitor offers CLI, config-as-code, or AI-agent integration. The entire space is dashboard-driven or service-driven.

**Unmet needs**:

- **Config-as-code**: Nobody provides version-controlled, diffable, auditable ad mediation configuration. Studios can't PR their tier changes.
- **Cross-mediator management**: Studios running LevelPlay for some apps and MAX for others have no single management layer. Bidlogic supports both, but as a black-box service.
- **AI-agent integration**: Zero competitors have MCP or AI-agent tooling. This is a novel distribution channel Admedi would own.
- **Audit trail**: No tool provides a historical record of what changed, when, and across which apps.

**Our opportunity**: The gap between "do it manually in dashboards" and "hire Bidlogic/Superscale to do it for you" is where Admedi lives. A self-serve developer tool that costs $50–150/mo and gives studios control over their own config with automation, version control, and AI integration.

---

## Our Positioning

### Positioning Statement

> For **ad monetization managers at indie and mid-size mobile game studios** who **waste hours per week clicking through mediation dashboards to sync configs across multiple apps and platforms**, **Admedi** is a **config-as-code tool** that **replaces 36+ dashboard clicks with one command**. Unlike **Bidlogic and Superscale** which require handing over control and paying revenue share, Admedi **gives you the automation while you keep full control of your configs** — open-source CLI, AI-agent-ready via MCP, and managed SaaS for teams that want zero ops.

### Competitive Edge

**Our unfair advantage**: The combination of config-as-code + open-source + MCP/AI integration is entirely novel in this space. Every competitor is either a black-box service (Bidlogic, Superscale) or abandoned (ironSource lib). We're building the Terraform for ad mediation — infrastructure-as-code that studios version control, diff, and apply programmatically.

**Why this matters to customers**: Studios keep full control. They see exactly what will change before it changes. They can review tier adjustments in a PR. They can roll back. They can integrate with their existing DevOps workflows. And for studios already using Claude Code or Cursor, the MCP tools drop into their AI workflow natively.

### Defensibility Assessment

| Moat Type | Strength | Notes |
|-----------|----------|-------|
| Technical | Medium | Adapter pattern + config engine is well-architected but replicable. First-mover in config-as-code gives head start. |
| Data/Network | Weak (growing) | No network effects at MVP. SaaS version could accumulate benchmarking data (anonymized eCPM comparisons) that creates data moat over time. |
| Brand/Trust | Medium | Open-source builds trust. Being the maintained replacement for ironSource lib gives credibility. "Powered by Creational.ai" adds legitimacy. |
| Switching costs | Medium | Once studios build their tier templates in YAML and integrate CLI into their workflow, switching is friction. Config history and audit logs create stickiness. |
| Cost advantage | Strong | ~$15/mo infrastructure cost means we can profitably serve at $49/mo with 70%+ gross margin. Revenue-share competitors can't compete on price for smaller studios. |

**Overall defensibility**: Medium — strong enough to build a business, especially with first-mover advantage in a niche where nobody else is building. The real moat develops over time as we accumulate adapters, integrations, and community trust.

---

## Target Customer Profile

### Ideal Customer (First 200 Users)

**Who they are**:

- **Role/Title**: Ad Monetization Manager, Growth Lead, or the studio founder who wears the ad ops hat
- **Company/Context**: Indie or mid-size mobile game studio, 5–50 employees, running 3–15 ad-supported apps across Android/iOS (some Amazon). Annual ad revenue $100K–$5M.
- **Demographics**: Technically capable (comfortable with CLI or would adopt an AI agent), data-driven, time-constrained

**Their problem**:

- **Pain point**: Every time they want to update country tiers, adjust floor prices, or add a new ad network instance, they're clicking through dashboard UIs repeatedly — the same changes, applied one-by-one across 6+ apps × 3 platforms = 18+ configuration surfaces. A portfolio-wide tier update takes 30–60 minutes of repetitive clicking.
- **Current solution**: Manual dashboard configuration in LevelPlay / MAX UI. Some may have internal Python scripts cobbled together from the abandoned ironSource library.
- **Why current solution fails**: Not scalable, no audit trail, no version control, high error rate (typos, missed apps), no way to preview changes before applying.

**Buying behavior**:

- **Trigger**: Adding a 4th or 5th app to their portfolio and realizing the manual config burden is multiplying. Or discovering a config mismatch across apps that cost them revenue.
- **Decision process**: Evaluates based on ease of setup, time saved, and whether the tool fits their existing workflow. Trial period is critical — needs to see value within 30 minutes of setup.
- **Willingness to pay**: $50–150/mo is a no-brainer vs. the monetization manager's time ($148K/yr avg salary). Even saving 4 hours/month justifies the cost.
- **Budget holder**: Studio founder or Head of Monetization. At this price point, it's a credit card purchase, not a procurement process.

### Where They Are

**Online communities**:

- r/gamedev — 2.5M+ members, active discussions on mobile game monetization
- r/androiddev, r/iOSProgramming — mobile developer communities
- TouchArcade forums — largest iPhone gaming community
- Buildbox community — mobile game dev and monetization discussions
- Unity forums — LevelPlay / ironSource integration discussions
- Discord servers: Mobile Games Dev, Indie Game Devs, various engine-specific servers

**Platforms**:

- GitHub — discover open-source tools, star repos, contribute
- Claude Code / Cursor / AI coding tools — MCP tool discovery
- LinkedIn — ad monetization manager professional network
- X/Twitter — gamedev and adtech conversations

**Content they consume**:

- PocketGamer.biz — mobile game industry news
- Business of Apps — mobile monetization trends and benchmarks
- Udonis blog — mobile marketing and monetization guides
- GameBiz Consulting blog — ad monetization best practices
- Tenjin blog — eCPM benchmarks, ad monetization reports

**Events/Conferences**:

- GDC (Game Developers Conference) — San Francisco, annually
- Pocket Gamer Connects — London (Jan 2026), San Francisco (Mar 2026), global series
- MAU (Mobile Apps Unlocked) — mobile growth conference
- Casual Connect / GamesBeat — indie/casual game focused

---

## Go-to-Market Strategy

### Recommended GTM Motion

**Primary approach**: Community-led + Product-led (open-source)

**Rationale**: The target audience is developers and technically-oriented ad ops people who discover tools through GitHub, Reddit, and word-of-mouth. An open-source CLI that solves a real pain point is the marketing asset. The product sells itself when someone runs `admedi audit` and sees config mismatches across their portfolio for the first time.

### Channel Prioritization

| Channel | Potential | Effort | Cost | Priority |
|---------|-----------|--------|------|----------|
| GitHub + open-source discovery | H | M | $ | 1 |
| r/gamedev + mobile dev subreddits | H | L | $ | 2 |
| PocketGamer.biz / Business of Apps content | M | M | $ | 3 |
| MCP tool directory / Claude ecosystem | M | L | $ | 4 |
| Pocket Gamer Connects conferences | M | H | $$ | 5 |
| LinkedIn outreach to monetization managers | M | M | $ | 6 |
| Cowork plugin marketplace | M | L | $ | 7 |

### Path to First 200 Users

**Phase 1: Seed (Users 1–20)**

- Dogfood on Shelf Sort portfolio (6 apps × 3 platforms). This is user #1 — us.
- Post launch announcement on r/gamedev with a real before/after: "We synced tier configs across 18 surfaces in 90 seconds instead of 45 minutes"
- Share the GitHub repo in Unity/LevelPlay forums with a clear README showing the abandoned ironSource lib replacement story
- Direct outreach to 10–15 studios we know in the mobile game space

**Phase 2: Validate (Users 21–50)**

- Publish Tenjin-style eCPM benchmark content (using Admedi's reporting pull) on PocketGamer.biz and Business of Apps
- Submit to MCP tool directories and Claude plugin marketplace
- Conference presence at PG Connects London (Jan 2026) or San Francisco (Mar 2026)
- Collect user feedback, iterate on config template format and CLI UX

**Phase 3: Scale (Users 51–200)**

- Launch SaaS tier with managed hosting, scheduled syncs, and audit dashboard
- Case study: "How [Studio X] eliminated config drift across 12 apps"
- MAX adapter launch — doubles the addressable market
- Cowork plugin launch — reaches non-technical monetization managers

### Distribution Shortcuts

**Potential partnerships**:

- **Tenjin / Singular**: Analytics platforms that could integrate Admedi's config management as a complementary feature
- **optAd360 / Bidlogic**: Bidlogic focuses on optimization, Admedi on configuration — potentially complementary rather than competitive

**Marketplaces/Platforms**:

- Claude Desktop plugin marketplace — first-mover in ad mediation AI tooling
- MCP tool registries — as MCP adoption grows, being listed early compounds

**Integration opportunities**:

- Unity Asset Store — package the CLI as a Unity editor extension
- GitHub Actions — CI/CD integration for "config deploy on merge"

---

## Pricing Strategy

### Competitive Pricing Landscape

| Competitor | Model | Price Point | Notes |
|------------|-------|-------------|-------|
| Bidlogic | Revenue share | Custom (% of uplift) | No published pricing. Premium positioning. |
| Superscale | Revenue share | % of uplift, no upfront | Full-service managed monetization |
| GameBiz Consulting | Consulting fees | Custom hourly/retainer | Consulting, not tooling |
| Tenjin (analytics, adjacent) | Subscription | $200/mo flat | MMP + ad revenue analytics |
| CAS.AI (mediation platform) | Usage-based | Not published | Mediation platform, not management tool |
| ironSource Python lib | Free (OSS) | $0 | Abandoned, no support |
| Terraform Cloud (analogy) | Per-resource subscription | Free → $0.10–$0.99/managed resource | Closest business model analogy |

### Recommended Pricing

**Model**: Tiered subscription, per-app pricing

**Rationale**: Per-app pricing directly aligns cost with value — the more apps a studio manages through Admedi, the more time they save. This mirrors Terraform's per-resource model and creates natural expansion revenue as studios grow their portfolios.

**Tiers**:

| Tier | Price | Includes | Target Customer |
|------|-------|----------|-----------------|
| **Open Source** | $0 | CLI, Python library, MCP server, local file storage, unlimited apps | Developers who self-host, contributors, evaluators |
| **Pro** | $49/mo (up to 10 apps) + $5/app beyond 10 | Managed SaaS hosting, Postgres persistence, config versioning, audit history, scheduled syncs, email alerts | Solo monetization managers, indie studios (3–10 apps) |
| **Scale** | $149/mo (up to 30 apps) + $4/app beyond 30 | Everything in Pro + multi-mediator support (LevelPlay + MAX), team access (3 seats), priority support, config rollback | Mid-size studios (10–30 apps), small teams |
| **Enterprise** | Custom | Everything in Scale + dedicated instance, SLA, SSO, unlimited seats, custom integrations, white-label option | Large publishers, agencies managing multiple studio portfolios |

### Unit Economics Assessment

**Revenue side**:

- Average revenue per user (ARPU): ~$75/mo (blended across Pro and Scale)
- Expected lifetime: 24 months (config tooling is sticky once integrated into workflow)
- Lifetime value (LTV): $1,800

**Cost side** (from Architecture doc):

- Infrastructure cost per tenant: ~$2–5/mo (shared App Runner + RDS, multi-tenant)
- Support/operational cost per tenant: ~$5/mo (at scale, mostly self-serve)
- Total cost per tenant: ~$7–10/mo

**Margins**:

- Gross margin: ~87–90%
- LTV:CAC target: 5:1+ (low CAC due to open-source-led discovery)
- Payback period: 1–2 months (credit card self-serve, no sales cycle)

**Assessment**: Viable — strong unit economics. Infrastructure costs are negligible due to shared multi-tenant architecture on existing Creational.ai AWS stack. The main cost is engineering time to build and maintain, not per-user infrastructure.

**Revenue targets**:

| Milestone | Users | MRR | ARR |
|-----------|-------|-----|-----|
| 6 months post-SaaS launch | 30 Pro + 5 Scale | $2,215 | $26,580 |
| 12 months | 80 Pro + 15 Scale | $6,155 | $73,860 |
| 24 months | 150 Pro + 40 Scale | $13,310 | $159,720 |

These are conservative targets. The multiplier effect kicks in when the MAX adapter ships (doubles addressable market) and the Cowork plugin launches (reaches non-technical buyers).

---

## Risks

| Risk | Category | Severity | Likelihood | Mitigation |
|------|----------|----------|------------|------------|
| LevelPlay or MAX builds native bulk config management | Competition | H | M | First-mover advantage, cross-mediator value prop. If one platform builds it, the other won't — cross-platform remains our edge. |
| Niche market too small to justify investment | Market | M | M | Dogfood first (our own portfolio). Open-source cost is near-zero. SaaS only needs 50 customers to cover costs. |
| Bidlogic expands into self-serve tooling | Competition | M | L | Bidlogic is positioned as premium managed optimization. Self-serve config tooling is a different product DNA. Unlikely pivot. |
| LevelPlay API deprecation or breaking changes | Technical | H | M | Version detection, support both v2 and v4. Unity's API churn is real but manageable with adapter isolation. |
| Low conversion from free to paid | Distribution | M | M | SaaS features (scheduled syncs, audit trail, team access) are genuinely valuable. Free tier is powerful but limited to local storage. |
| MCP/AI-agent adoption slower than expected | Timing | L | M | MCP is a bonus channel, not the primary. CLI and open-source distribution work regardless of MCP adoption. |
| MAX API access restricted or requires partnership | Technical | M | M | AppLovin published the Ad Unit Management API publicly. If gated, deprioritize MAX and focus on LevelPlay + AdMob. |

### Critical Risks

No High/High risks identified. The highest-impact risk is platform competition (LevelPlay/MAX building native tools), but this has medium likelihood — neither has done so in 3+ years of having the APIs. The cross-mediator value prop also insulates against single-platform competition.

---

## Open Questions

**Must answer before proceeding**:

- [ ] MAX API access verification — is the Ad Unit Management API fully public or does it require partnership approval?
- [ ] Validate pricing sensitivity with 5–10 studios outside our network before SaaS launch

**Should validate during early traction**:

- [ ] Is the "per app" pricing model intuitive to buyers, or would flat tiers (Small/Medium/Large) convert better?
- [ ] What's the actual time-to-value for a new user? Can they go from install to first audit in <30 minutes?
- [ ] Do non-technical monetization managers actually adopt Cowork plugin, or is this developer-only?

**Nice to know**:

- [ ] Would studios pay for anonymized eCPM benchmarking data (aggregate performance comparison across Admedi users)?
- [ ] Is there demand for a "config marketplace" where studios share tier templates?

---

## Recommendation

### Verdict: GO

### Reasoning

- **Genuine whitespace**: Zero config-as-code tools exist for ad mediation. The gap between "manual dashboards" and "hire a managed service" is wide open, and nobody is building in it.
- **We eat our own dogfood**: Shelf Sort portfolio (6 apps × 3 platforms) is the perfect first customer. We build what we need, then offer it to others. This is the strongest possible validation path.
- **Low downside risk**: The open-source core has near-zero marginal cost. SaaS only needs ~50 paying customers to cover infrastructure. If the market doesn't materialize beyond us, we still have the internal tool we need.
- **Strong unit economics**: 87–90% gross margin, ~$7–10/mo cost per tenant, $49–149/mo price points. The math works at any reasonable scale.
- **Timing convergence**: Abandoned ironSource lib (vacuum) + mature mediation APIs (capability) + MCP protocol adoption (distribution) = rare alignment of market conditions.

### Key success factors

- **Ship the open-source CLI fast**: The working product is the marketing asset. Speed to first public release matters more than feature completeness.
- **Nail the config template format**: YAML template design determines the product's usability. It needs to be intuitive enough that a monetization manager can read it, and powerful enough that it covers the 80% use case.
- **MAX adapter within 6 months of launch**: Cross-mediator support is the moat. LevelPlay-only is a strong MVP, but the full value prop requires at least two adapters.

**What to watch**:

- GitHub stars and CLI downloads in first 3 months (target: 50 stars, 200+ installs)
- Free-to-paid conversion rate (target: >5% within first 3 months of SaaS launch)
- Config template reuse rate (are users creating templates, or just doing one-off operations?)
- Time-to-value for new users (target: first audit report within 30 minutes of install)

**Suggested next steps**:

1. Complete Architecture doc review and move to Roadmap (Design Stage 3)
2. Build LevelPlay adapter MVP and ConfigEngine — dogfood on Shelf Sort portfolio
3. Open-source launch with CLI + GitHub repo
4. Launch SaaS tier 2–3 months after open-source proves the tool works
5. MAX adapter research and development
6. Cowork plugin packaging

---

## Sources

- [Growth Market Reports — Mobile Ad Mediation Platform Market](https://growthmarketreports.com/report/mobile-ad-mediation-platform-market)
- [Growth Market Reports — Ad Mediation Platforms for Games Market](https://growthmarketreports.com/report/ad-mediation-platforms-for-games-market)
- [Fortune Business Insights — Mobile Advertising Market](https://www.fortunebusinessinsights.com/mobile-advertising-market-102496)
- [Statista — In-App Advertising Worldwide](https://www.statista.com/outlook/amo/advertising/in-app-advertising/worldwide)
- [6sense — Mobile Ad Mediation Software Market Share](https://6sense.com/tech/mobile-ad-mediation)
- [Bidlogic — Ad Mediation Automation](https://bidlogic.io/)
- [Superscale — Ad Mediation Platform](https://superscale.com/ad-mediation-platform/)
- [GameBiz Consulting — Ad Monetization Services](https://www.gamebizconsulting.com/ad-monetization)
- [AppLovin — MAX Ad Unit Management API](https://www.applovin.com/blog/automate-your-monetization-with-maxs-new-ad-unit-management-api/)
- [Tenjin — Pricing](https://tenjin.com/pricing/)
- [Glassdoor — Monetization Manager Salary](https://www.glassdoor.com/Salaries/monetization-manager-salary-SRCH_KO0,20.htm)
- [Spacelift — Terraform Cloud Pricing](https://spacelift.io/blog/terraform-cloud-pricing)
- [Tracxn — Bidlogic Company Profile](https://tracxn.com/d/companies/bidlogic/__ANi7mm6eL-0f7f6Oc-hfO6LWd_3SjZ9ouHLY_Y2Znek)
- [Singular — Ad Mediation Revenue](https://www.singular.net/blog/ad-mediation-revenue/)
- [PocketGamer.biz — Events](https://www.pocketgamer.biz/events/)
- [Business of Apps — Mobile Advertising Market](https://www.blog.udonis.co/advertising/mobile-advertising-market)
- [Udonis — Top Ad Mediation Platforms](https://www.blog.udonis.co/mobile-marketing/top-ad-mediation-platforms-mobile-apps-games)
