# Admedi — Roadmap

**Version:** 1.0
**Vision**: Config-as-code tool for ad mediation — manage country tiers, waterfalls, and instances across your entire app portfolio from one YAML template, via CLI, MCP, or SaaS.

**Related Documents**:
- [Architecture Doc](./admedi-architecture.md)
- [Vision Doc](./admedi-vision.md)
- [Market Research](./admedi-market-research.md)
- [API Reference](./levelplay-api-reference.md)

**Strategic Approach**: Internal Tool (Mochibits) → Open Source → SaaS → Multi-Mediator → AI Intelligence

---

## Milestone Progression

```
┌──────────────────────────────────────────────────────────────────────────────────┐
│                           MILESTONE PROGRESSION                                   │
├──────────────────────────────────────────────────────────────────────────────────┤
│                                                                                   │
│  Core                  Open Source           SaaS                Multi-Mediator   │
│  ═════                 ══════════            ════                ══════════════   │
│                                                                                   │
│  Internal Tool  ────▶  Community    ────▶   Hosted      ────▶  Cross-Platform   │
│  Mochibits             Launch               Offering            Management        │
│                                                                                   │
│  ┌──────────────┐     ┌──────────────┐     ┌──────────────┐   ┌──────────────┐  │
│  │ LevelPlay    │     │ README +     │     │ Postgres     │   │ MAX Adapter  │  │
│  │ Adapter      │     │ Docs         │     │ Adapter      │   │      ↓       │  │
│  │      ↓       │     │      ↓       │     │      ↓       │   │ Unified      │  │
│  │ ConfigEngine │     │ SQLite       │     │ Multi-Tenant │   │ Config       │  │
│  │      ↓       │     │ Adapter      │     │ Auth         │   │ Templates    │  │
│  │ CLI + MCP    │     │      ↓       │     │      ↓       │   │      ↓       │  │
│  │      ↓       │     │ CI/CD +      │     │ Scheduled    │   │ Adapter      │  │
│  │ Local File   │     │ Tests        │     │ Syncs        │   │ Auto-Detect  │  │
│  │ Storage      │     │      ↓       │     │      ↓       │   │      ↓       │  │
│  │      ↓       │     │ Community    │     │ Audit        │   │ Cross-       │  │
│  │ Shelf Sort   │     │ Feedback     │     │ Dashboard    │   │ Mediator     │  │
│  │ Dogfood      │     │              │     │              │   │ Reporting    │  │
│  └──────────────┘     └──────────────┘     └──────────────┘   └──────────────┘  │
│                                                                                   │
│  OUTCOME:              OUTCOME:             OUTCOME:           OUTCOME:           │
│  • Shelf Sort fully    • GitHub repo live   • Paying SaaS      • MAX + LevelPlay │
│    managed via CLI     • 50 GitHub stars      customers          in one config    │
│  • 18 surfaces synced  • Community PRs      • $2K+ MRR         • Doubles TAM     │
│  • Config drift = 0    • SQLite option      • Scheduled syncs   • Cross-platform  │
│  • MCP tools working   • Battle-tested CI   • Audit history       audit           │
│                                                                                   │
│                                                                                   │
│                        ┌──────────────┐                                           │
│                        │ Intelligence │  (Longer Play — post product-market fit)  │
│                        │ ═══════════  │                                           │
│                        │ AI Recs      │                                           │
│                        │      ↓       │                                           │
│                        │ A/B Testing  │                                           │
│                        │      ↓       │                                           │
│                        │ Benchmarks   │                                           │
│                        └──────────────┘                                           │
│                                                                                   │
└──────────────────────────────────────────────────────────────────────────────────┘
```

---

## Core

**[📄 Detailed Plan](../core-milestone-spec.md)**

**Status**: Planning

### Goal

Build Admedi as a working internal tool for Mochibits' Shelf Sort portfolio. Implement the LevelPlay adapter, ConfigEngine pipeline, CLI, and MCP server — all backed by local file storage. Validate that the config-as-code approach works by syncing tier configurations across all 6 apps × 3 platforms (18 configuration surfaces) from a single YAML template. This is the foundation everything else builds on — if the core engine doesn't work for us, nothing else matters.

### Architecture Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                    CORE MILESTONE                            │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌─────────┐  ┌─────────┐  ┌──────────┐                    │
│  │  CLI    │  │ FastMCP │  │ Python   │                    │
│  │ (typer) │  │ Server  │  │ Library  │                    │
│  └────┬────┘  └────┬────┘  └────┬─────┘                    │
│       └─────────────┼───────────┘                           │
│                     │                                        │
│       ┌─────────────▼──────────────┐                        │
│       │       ConfigEngine         │                        │
│       │  Loader → Differ → Applier │                        │
│       └─────────────┬──────────────┘                        │
│              ┌──────┴──────┐                                │
│              │             │                                │
│    ┌─────────▼───┐  ┌─────▼─────────┐                      │
│    │  LevelPlay  │  │  Local File   │                      │
│    │  Adapter    │  │  Storage      │                      │
│    │  (Groups,   │  │  (.admedi/)  │                      │
│    │  Instances, │  └───────────────┘                      │
│    │  Placements,│                                          │
│    │  Reporting) │                                          │
│    └─────────────┘                                          │
│           │                                                  │
│    ┌──────▼──────┐                                          │
│    │  LevelPlay  │                                          │
│    │  REST API   │                                          │
│    │  (platform. │                                          │
│    │  ironsrc.   │                                          │
│    │  com)       │                                          │
│    └─────────────┘                                          │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### What Gets Built

**Phase 1: Foundation**
- Project scaffolding: repo structure, pyproject.toml, pydantic models for all 9 core entities (App, TierTemplate, Group, WaterfallConfig, Instance, Placement, SyncLog, ConfigSnapshot, Credential)
- LevelPlay adapter: OAuth 2.0 Bearer auth (secretKey + refreshToken → JWT, auto-refresh), httpx async client
- LevelPlay adapter: Read endpoints — `list_apps()`, `get_groups()`, `get_instances()`, `get_placements()`, `get_reporting()`

**Phase 2: ConfigEngine**
- YAML tier template format design and Loader implementation (parse YAML → normalized TierTemplate objects)
- Differ: compare local TierTemplate against remote Groups per app, produce DiffReport (added/removed countries, floor price changes, new groups needed)
- Applier: execute DiffReport by calling LevelPlay adapter write methods (`create_group`, `update_group`, `delete_group`, `create_instances`, `update_instances`, `delete_instance`)

**Phase 3: Interfaces**
- CLI via typer: `admedi sync-tiers`, `admedi audit`, `admedi revenue`, `admedi manage-instances`, `admedi status`
- MCP server via FastMCP: generic tool names (`get_groups`, `sync_tiers`, `audit`, `revenue_check`, `manage_instances`, `status`)
- Local file storage adapter: YAML configs + JSON sync logs in `.admedi/` directory

**Phase 4: Dogfood**
- Create Shelf Sort tier template YAML from current LevelPlay dashboard config
- Run `admedi audit` across all 18 surfaces — verify it catches the known mismatches
- Run `admedi sync-tiers` — verify it applies the template correctly
- End-to-end MCP test: use Claude Code to query and manage the portfolio via MCP tools

### Success Metrics

**Functionality**:
- **API Coverage**: 100% of LevelPlay management endpoints (Groups v4, Instances v3, Placements v1, Reporting v1)
- **Config Sync**: All 18 surfaces synced from one YAML template in < 2 minutes
- **Audit Accuracy**: Detects all known config mismatches between template and live config
- **MCP Tools**: All 6 core tools functional and callable from Claude Code

**Reliability**:
- **Auth**: OAuth token auto-refresh works across multi-hour sessions
- **Rate Limits**: Graceful handling with exponential backoff (never hits hard 429 errors)
- **Error Handling**: Atomic per-app operations — partial failure on one app doesn't corrupt others

**Validation**:
- **Dogfood**: Shelf Sort portfolio (6 apps × 3 platforms) fully managed via CLI for 2+ weeks with zero manual dashboard touches

### Key Outcomes

✅ Shelf Sort portfolio fully managed via config-as-code — no more dashboard clicking
✅ ConfigEngine pipeline (Loader → Differ → Applier) proven on real production data
✅ LevelPlay adapter battle-tested against live API with real credentials
✅ MCP tools working — Admedi usable via Claude Code conversation
✅ YAML tier template format validated against real-world mediation config complexity

### Why Internal-First?

- **Eat our own dogfood**: We have 18 real configuration surfaces that need this tool today. Building for ourselves first means we catch real issues before anyone else hits them.
- **Zero marketing risk**: No need to convince anyone to try an unproven tool. We are the customer.
- **Fast iteration**: No backward compatibility concerns, no user communications, no breaking changes drama. Move fast.
- **Validates the core thesis**: If the config-as-code approach doesn't save us time on 18 surfaces, it won't save anyone time. Prove it works before going public.

---

## Open Source

**[📄 Detailed Plan](../opensource-milestone-spec.md)**

**Status**: Planning

### Goal

Package the battle-tested internal tool for public consumption. Clean up the repo, write comprehensive docs, add SQLite storage adapter, set up CI/CD, and launch on GitHub. The goal is to become the maintained replacement for the abandoned ironSource Python library and the go-to open-source tool for ad mediation config management. Core milestone validates the engine works; this milestone validates that other studios can use it.

### Architecture Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                  OPEN SOURCE MILESTONE                       │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌──────────────────────────────────────────┐               │
│  │          GitHub: creational-ai/admedi   │               │
│  │                                          │               │
│  │  README.md    CONTRIBUTING.md            │               │
│  │  Apache-2.0   /ee (commercial)           │               │
│  │  CI/CD        pyproject.toml             │               │
│  └──────────────────┬───────────────────────┘               │
│                     │                                        │
│       ┌─────────────┼─────────────┐                         │
│       │             │             │                         │
│  ┌────▼────┐  ┌─────▼─────┐  ┌───▼───────┐                │
│  │ Local   │  │  SQLite   │  │ Postgres  │                │
│  │ File    │  │  Adapter  │  │ (stub/    │                │
│  │(default)│  │  (new)    │  │  /ee)     │                │
│  └─────────┘  └───────────┘  └───────────┘                │
│                                                              │
│  ┌──────────────────────────────────────────┐               │
│  │  GitHub Actions CI/CD                    │               │
│  │  pytest + pytest-asyncio + mypy + ruff   │               │
│  │  Mock adapter for CI (no real API calls) │               │
│  └──────────────────────────────────────────┘               │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### What Gets Built

**Phase 1: Code Cleanup & Docs**
- README with clear value prop, quick start (5-minute setup), and real before/after examples
- CONTRIBUTING.md with adapter development guide
- Inline docstrings and type hints on all public interfaces
- Example YAML tier templates for common setups (casual game 4-tier, hyper-casual 2-tier)

**Phase 2: SQLite Storage Adapter**
- SQLite adapter implementing StorageAdapter interface (save_config, load_config, save_sync_log, list_sync_history, save_snapshot)
- Migration scripts for schema setup
- Zero external dependency — Python stdlib `sqlite3`

**Phase 3: CI/CD & Testing**
- GitHub Actions: lint (ruff), type check (mypy), unit tests (pytest), integration tests (mock adapter)
- Mock LevelPlay adapter for CI — tests run without real API credentials
- Test coverage target: >80% on core engine, >60% on adapters
- Git tag-based versioning (`v1.0.0`, `v1.1.0`)

**Phase 4: Community Launch**
- Apache-2.0 LICENSE + `/ee` directory with commercial license stub
- GitHub release with changelog
- r/gamedev launch post with real Shelf Sort before/after metrics
- Unity/LevelPlay forum posts — "maintained replacement for ironSource Python lib"

### Success Metrics

**Adoption**:
- **GitHub Stars**: 50 within first 3 months
- **Installs**: 200+ `pip install` from GitHub within 3 months
- **Issues/PRs**: 10+ community-filed issues or PRs within 3 months

**Quality**:
- **Test Coverage**: >80% core engine, >60% adapters
- **CI Green**: All tests pass on every PR
- **Zero Breaking Changes**: Semver-compliant from v1.0.0

**Community**:
- **External Contributors**: 3+ unique contributors within 6 months
- **Documentation Completeness**: Every public API method documented with docstring and usage example

### Key Outcomes

✅ Public GitHub repo with clean README and real usage examples
✅ SQLite storage adapter available for self-hosted query capability
✅ CI/CD pipeline ensures quality on every commit
✅ Community validation — other studios are using the tool and filing issues
✅ Open-core licensing structure (Apache-2.0 + `/ee`) established

### Why Open Source Before SaaS?

- **Distribution engine**: Open-source projects get discovered organically via GitHub search, Reddit, and word-of-mouth. This is free marketing.
- **Trust signal**: Studios are trusting us with their mediation API credentials. Open-source code is auditable — that matters.
- **Community feedback**: Real users filing real issues makes the tool better before we charge for it.
- **Recruitment funnel**: Open-source contributors → SaaS beta testers → paying customers. The funnel builds itself.

---

## SaaS

**[📄 Detailed Plan](../saas-milestone-spec.md)**

**Status**: Planning

### Goal

Launch the Creational.ai hosted version of Admedi. Build the Postgres storage adapter, multi-tenant auth, scheduled syncs, and audit dashboard. Deploy on existing AWS infrastructure (App Runner + RDS). Convert open-source users who want managed hosting and zero-ops into paying SaaS customers. Target: $2K+ MRR within 6 months of SaaS launch.

### Architecture Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                      SAAS MILESTONE                          │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌──────────────────────────────────────────┐               │
│  │         Creational.ai SaaS               │               │
│  │                                          │               │
│  │  ┌─────────┐  ┌───────────┐             │               │
│  │  │ FastMCP │  │ REST API  │             │               │
│  │  │ Server  │  │ (tenant   │             │               │
│  │  │         │  │  mgmt)    │             │               │
│  │  └────┬────┘  └─────┬─────┘             │               │
│  │       └──────┬───────┘                   │               │
│  │              │                           │               │
│  │  ┌───────────▼────────────┐             │               │
│  │  │  /ee (commercial)      │             │               │
│  │  │  Multi-tenant auth     │             │               │
│  │  │  Scheduled syncs       │             │               │
│  │  │  Audit dashboard       │             │               │
│  │  │  Usage metering        │             │               │
│  │  └───────────┬────────────┘             │               │
│  │              │                           │               │
│  │  ┌───────────▼────────────┐             │               │
│  │  │  Postgres Adapter      │             │               │
│  │  │  (asyncpg → RDS)       │             │               │
│  │  │  Multi-tenant isolation│             │               │
│  │  │  Config versioning     │             │               │
│  │  │  Full audit history    │             │               │
│  │  └────────────────────────┘             │               │
│  └──────────────────────────────────────────┘               │
│                     │                                        │
│  ┌──────────────────▼───────────────────────┐               │
│  │  AWS (Existing Creational.ai Stack)      │               │
│  │  App Runner + RDS Postgres               │               │
│  └──────────────────────────────────────────┘               │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### What Gets Built

**Phase 1: Postgres Adapter**
- Postgres storage adapter implementing StorageAdapter interface via asyncpg
- Multi-tenant schema isolation (tenant_id on all tables)
- Config versioning — every save creates a new version, rollback to any previous version
- Migration scripts (alembic or raw SQL)

**Phase 2: Multi-Tenant Auth & Onboarding**
- API key per tenant (matches Creational.ai auth patterns)
- Tenant onboarding flow: create account → store mediation credentials (encrypted) → discover apps
- Credential encryption at rest (AES-256 via RDS encryption)

**Phase 3: SaaS Features (/ee)**
- Scheduled syncs: cron-style automatic tier sync + drift detection with email alerts
- Audit dashboard: web UI showing sync history, config diffs over time, drift detection timeline
- Usage metering: track apps managed per tenant for billing

**Phase 4: Launch & Pricing**
- Stripe integration for subscription billing
- Landing page on Creational.ai with pricing tiers (Free / Pro $49/mo / Scale $149/mo / Enterprise custom)
- Cowork plugin packaging — one-click install for Claude Desktop users
- Plugin skills: setup, sync-tiers, audit-config, revenue-check, manage-instances

### Success Metrics

**Revenue**:
- **Paying Customers**: 30+ Pro + 5+ Scale within 6 months
- **MRR**: $2,000+ within 6 months of SaaS launch
- **Churn**: < 5% monthly

**Product**:
- **Scheduled Syncs**: Working reliably for all tenants (zero missed syncs)
- **Audit Dashboard**: Accessible and useful (>50% of SaaS customers use it weekly)
- **Onboarding Time**: New tenant from signup to first audit < 15 minutes

**Operations**:
- **Infrastructure Cost**: < $25/mo total (shared App Runner + RDS)
- **Uptime**: 99.5%+ (App Runner auto-scaling handles this)

### Key Outcomes

✅ Paying SaaS customers generating recurring revenue
✅ Postgres adapter proven for multi-tenant workloads
✅ Scheduled syncs and audit dashboard differentiate SaaS from free tier
✅ Cowork plugin live — non-technical monetization managers can use Admedi
✅ Revenue path validated — unit economics work in practice

### Why SaaS After Open Source?

- **Conversion funnel**: Open-source users who hit the ceiling of local file storage are natural SaaS leads. They already trust the tool.
- **Feature differentiation**: Scheduled syncs, audit history, and team access are features that only make sense in a hosted context. Clear upgrade path.
- **Existing infrastructure**: App Runner + RDS already run Mission Control and Video Professor. SaaS is an incremental deployment, not a new stack.
- **Revenue validation**: Open-source proves the tool works. SaaS proves studios will pay for it.

---

## Multi-Mediator

**[📄 Detailed Plan](../multimediator-milestone-spec.md)**

**Status**: Planning

### Goal

Add the MAX adapter as the second mediation platform, proving the adapter architecture works across mediators. Build unified config templates that can target different mediators per app (apps 1-3 on LevelPlay, apps 4-6 on MAX). This doubles the addressable market and delivers on the core vision — Terraform for ad mediation, not just Terraform for LevelPlay.

### Architecture Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                 MULTI-MEDIATOR MILESTONE                     │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│       ┌─────────────────────────────────┐                   │
│       │       ConfigEngine              │                   │
│       │  Loader → Differ → Applier      │                   │
│       └──────────────┬──────────────────┘                   │
│                      │                                       │
│            ┌─────────┼─────────┐                            │
│            │         │         │                            │
│  ┌─────────▼───┐  ┌──▼──────┐  ┌──▼───────────┐           │
│  │  LevelPlay  │  │  MAX    │  │  (future)    │           │
│  │  Adapter    │  │ Adapter │  │  AdMob       │           │
│  │  (existing) │  │  (new)  │  │  Adapter     │           │
│  └─────────────┘  └─────────┘  └──────────────┘           │
│                                                              │
│  ┌──────────────────────────────────────────┐               │
│  │  Unified YAML Template                   │               │
│  │                                          │               │
│  │  apps:                                   │               │
│  │    shelf-sort-android:                   │               │
│  │      mediator: levelplay                 │               │
│  │      tier_template: casual-4tier         │               │
│  │    puzzle-rush-android:                  │               │
│  │      mediator: max                       │               │
│  │      tier_template: casual-4tier         │               │
│  └──────────────────────────────────────────┘               │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### What Gets Built

**Phase 1: MAX Adapter**
- Research MAX Ad Unit Management API access and authentication
- MAX adapter implementing MediationAdapter interface (authenticate, list_apps, get_groups, create_group, update_group, get_instances, get_reporting)
- MAX-specific normalizers: translate MAX API format ↔ Admedi internal models

**Phase 2: Unified Config Templates**
- Extend YAML template format with `mediator` field per app
- ConfigEngine auto-detects mediator per app and routes to correct adapter
- Cross-mediator audit: "App X on LevelPlay has Tier 2 = [UK, AU, CA, DE, JP, NZ, KR, TW] but App Y on MAX has Tier 2 = [UK, AU, CA, DE, JP, NZ]" — catches drift across platforms

**Phase 3: Cross-Mediator Reporting**
- Unified reporting pull across LevelPlay and MAX
- Normalized eCPM comparison: "Your Tier 2 eCPM on LevelPlay is $9.20 vs $10.40 on MAX for the same countries"
- Portfolio-wide revenue dashboard combining both mediators

### Success Metrics

**Adapter Quality**:
- **MAX API Coverage**: Core endpoints (groups, instances, reporting) functional
- **Cross-Mediator Sync**: Single `admedi sync-tiers` command applies template to both LevelPlay and MAX apps
- **Audit Accuracy**: Cross-platform drift detection catches all mismatches

**Market**:
- **Addressable Market**: Doubles — studios using MAX can now adopt Admedi
- **New Users**: 20+ MAX-only or mixed-mediator users within 3 months of adapter launch

### Key Outcomes

✅ Two mediation adapters working — adapter pattern proven
✅ Unified config template manages mixed-mediator portfolios
✅ Cross-mediator reporting gives studios apples-to-apples eCPM comparison
✅ Addressable market doubled

### Why Multi-Mediator After SaaS?

- **Revenue-funded**: SaaS revenue justifies the engineering investment in a second adapter
- **Customer-driven**: SaaS customers will tell us exactly which mediator they need next. Build what they ask for, not what we guess.
- **Proves the architecture**: The adapter pattern was designed for this moment. If it works cleanly, the architecture is validated. If it's painful, we learn what to refactor before AdMob.
- **Competitive moat**: Cross-mediator management is the one thing nobody else does. LevelPlay tools only manage LevelPlay. MAX tools only manage MAX. We manage both from one config.

---

## Intelligence (Longer Play)

**[📄 Detailed Plan](../intelligence-milestone-spec.md)**

**Status**: Future (post product-market fit)

### Goal

Build the AI-powered intelligence layer on top of the config management foundation. Once all mediation data flows through Admedi — eCPMs, fill rates, revenue, config history — Claude can reason about it and propose data-backed optimizations. This is where Admedi evolves from infrastructure tooling into an intelligent monetization copilot. Only pursue after 200+ active users validate the core product.

### Architecture Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                  INTELLIGENCE MILESTONE                       │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌──────────────────────────────────────────┐               │
│  │           AI Recommendation Engine       │               │
│  │                                          │               │
│  │  Performance Data ──▶ Analysis ──▶ Thesis│               │
│  │                                          │               │
│  │  "Taiwan eCPM $8.20 vs Tier 3 avg       │               │
│  │   $3.40 — promote to Tier 2"            │               │
│  └──────────────────┬───────────────────────┘               │
│                     │                                        │
│  ┌──────────────────▼───────────────────────┐               │
│  │           A/B Test Framework             │               │
│  │                                          │               │
│  │  Propose ──▶ Approve ──▶ Execute         │               │
│  │                  ──▶ Measure ──▶ Learn   │               │
│  │                                          │               │
│  │  Test group: Apps 1-3 (change applied)   │               │
│  │  Control:    Apps 4-6 (no change)        │               │
│  │  Duration:   14 days                     │               │
│  │  Metric:     eCPM delta + revenue delta  │               │
│  └──────────────────┬───────────────────────┘               │
│                     │                                        │
│  ┌──────────────────▼───────────────────────┐               │
│  │        Anonymized Benchmarks             │               │
│  │                                          │               │
│  │  Aggregate eCPM data across all Admedi │               │
│  │  users (opt-in). "Your Tier 2 casual    │               │
│  │  eCPM is $9.20 — market avg is $10.40"  │               │
│  └──────────────────────────────────────────┘               │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### What Gets Built

**Phase 1: AI Recommendations (Layer 2)**
- Performance analysis engine: aggregate eCPM, fill rate, revenue data per tier/country/network
- Recommendation generator: identify underperforming countries, suggest tier promotions/demotions with supporting data
- MCP tool: `get_recommendations` returns prioritized list of suggested changes with rationale

**Phase 2: A/B Test Framework (Layer 3)**
- Test definition: specify test group (subset of apps), control group, change to apply, duration, success metric
- Automated execution: apply change to test group, hold control group, measure over defined period
- Results analysis: statistical significance check, revenue impact calculation, rollback if negative

### Success Metrics

**Intelligence**:
- **Recommendation Accuracy**: >60% of accepted recommendations produce measurable eCPM improvement
- **A/B Test Completion**: Studios run 5+ A/B tests within first 3 months of feature launch

**Data Network**:
- **Benchmark Participation**: >30% of SaaS users opt into anonymized benchmarking
- **Benchmark Utility**: Users who access benchmarks retain at 2x rate of those who don't

### Key Outcomes

✅ AI-powered tier optimization recommendations backed by real performance data
✅ A/B testing framework closes the loop: propose → approve → execute → measure → learn
✅ Anonymized benchmarks create data network effect — moat deepens with scale
✅ Studios build institutional knowledge from documented test results

### Why Intelligence Last?

- **Data dependency**: AI recommendations require months of historical config + performance data. Can't recommend tier changes without knowing how current tiers perform.
- **Trust requirement**: Studios need to trust Admedi with config management before they'll trust it with optimization suggestions. Build trust through reliability first.
- **200-user threshold**: Anonymized benchmarks only become valuable with sufficient scale. Premature launch = useless data.
- **Revenue justification**: This is the premium SaaS differentiator — higher tier pricing or revenue share model. Only invest when the base product has proven PMF.

---

## Strategic Decisions

### Why This Milestone Order?

**Core First**:
- We need the tool ourselves today — Shelf Sort has 18 config surfaces that are managed manually
- Building for ourselves means fast iteration, real-world validation, and zero adoption risk
- Every subsequent milestone depends on a working ConfigEngine + LevelPlay adapter

**Open Source Second**:
- Open-source is the distribution engine — free marketing through GitHub discovery
- Community feedback makes the tool better before we charge for it
- Establishes credibility as the maintained replacement for the abandoned ironSource Python library
- Apache-2.0 licensing builds trust (studios are handing over API credentials)

**SaaS Third**:
- Revenue path must be validated after the tool proves useful
- Natural conversion: open-source users who outgrow local file storage → SaaS
- SaaS features (scheduled syncs, audit dashboard, team access) are the upgrade incentive
- Builds on existing Creational.ai AWS infrastructure — incremental cost, not new stack

**Multi-Mediator Fourth**:
- Revenue-funded by SaaS — the engineering investment is justified by paying customers
- Customer-driven prioritization — SaaS users tell us which mediator they need next
- Proves the adapter architecture works (the moment of truth for the OOP design)
- Competitive moat — cross-mediator config management is unique in the market

**Intelligence Fifth (Longer Play)**:
- Requires accumulated data (months of config + performance history)
- Requires user trust (prove reliability before suggesting optimizations)
- Requires scale (benchmarks need 200+ users to be meaningful)
- This is the premium SaaS differentiator that justifies higher pricing

---

## Success Criteria

### Core
- [ ] LevelPlay adapter covers 100% of management endpoints
- [ ] ConfigEngine syncs 18 surfaces from one YAML template in < 2 minutes
- [ ] Audit command detects all config mismatches
- [ ] MCP tools callable from Claude Code
- [ ] Shelf Sort managed via CLI for 2+ weeks with zero manual dashboard touches

### Open Source
- [ ] GitHub repo live with README, Apache-2.0 license, contributing guide
- [ ] 50 GitHub stars within 3 months
- [ ] CI/CD green on every PR (lint, type check, tests)
- [ ] SQLite storage adapter functional
- [ ] 3+ external contributors within 6 months

### SaaS
- [ ] 30+ paying customers within 6 months of SaaS launch
- [ ] $2K+ MRR
- [ ] Scheduled syncs working reliably for all tenants
- [ ] Cowork plugin live and installable
- [ ] < $25/mo infrastructure cost

### Multi-Mediator
- [ ] MAX adapter functional (core endpoints)
- [ ] Single YAML template manages mixed LevelPlay + MAX portfolio
- [ ] Cross-mediator audit detects drift across platforms
- [ ] 20+ MAX or mixed-mediator users within 3 months

### Long-Term
- [ ] 200+ active users (free + paid)
- [ ] AI recommendations producing measurable eCPM improvement
- [ ] Anonymized benchmarks creating data network effect
- [ ] Self-sustaining SaaS revenue covering all development costs
- [ ] 3+ mediation adapters (LevelPlay, MAX, AdMob)

---

## Next Steps

**Current Status**: Vision (v1.7), Architecture (v1.0), Market Research (v1.0), and Roadmap (v1.0) complete. Core Milestone Spec and Task Spec complete. Ready for implementation.

**Next Action**: Begin implementation via dev skill — start with Task: Project Foundation

**Detailed Plans**:
- [📄 Core Milestone Spec](../core-milestone-spec.md) — Complete
- [📄 Core Task Spec](../core-task-spec.md) — Complete
- [📄 Open Source](../opensource-milestone-spec.md) — To be created
- [📄 SaaS](../saas-milestone-spec.md) — To be created
- [📄 Multi-Mediator](../multimediator-milestone-spec.md) — To be created
- [📄 Intelligence](../intelligence-milestone-spec.md) — To be created
