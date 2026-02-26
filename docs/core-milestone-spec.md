# Admedi — Core Milestone Spec

**Status**: Planning
**Parent Document**: [Roadmap](./references/admedi-roadmap.md)
**Architecture Reference**: [Architecture Doc](./references/admedi-architecture.md)

---

## Executive Summary

This milestone builds Admedi as a working internal tool for Mochibits' Shelf Sort portfolio. We implement the LevelPlay adapter, ConfigEngine pipeline (Loader → Differ → Applier), CLI, MCP server, and local file storage — then dogfood it on all 6 apps × 3 platforms (18 configuration surfaces). The goal is to replace 45+ minutes of repetitive dashboard clicking with a single `admedi sync-tiers` command.

This is the foundation everything else builds on. The LevelPlay adapter proves we can talk to the API reliably. The ConfigEngine proves config-as-code works for ad mediation. The CLI and MCP server prove the interfaces are usable. The dogfood period proves the tool actually saves time on a real portfolio. If this milestone fails, nothing else matters — no open-source launch, no SaaS, no multi-mediator.

**Key Principle**: Build for Mochibits first — solve our own problem with production-grade quality, then generalize.

---

## Goal

Build and validate the core Admedi engine as an internal tool for Mochibits' Shelf Sort portfolio. Implement the LevelPlay adapter, ConfigEngine, CLI, MCP server, and local file storage. Validate by managing all 18 configuration surfaces from a single YAML template for 2+ weeks with zero manual dashboard touches.

**What This Milestone Proves**:
- Config-as-code works for ad mediation — YAML templates can represent real-world tier configurations
- The LevelPlay adapter handles OAuth auth, rate limits, and all management endpoints reliably
- The Differ produces accurate diffs between local templates and remote configs
- The Applier correctly pushes changes without corrupting existing configs
- The CLI and MCP interfaces are usable enough for daily ad ops work

**What This Milestone Does NOT Include**:
- Open-source packaging (no README, no contributing guide, no public repo)
- SaaS hosting or multi-tenant support
- MAX or AdMob adapters
- SQLite or Postgres storage adapters
- AI optimization recommendations (Layer 2/3)
- Cowork plugin or skills
- Automated testing infrastructure (CI/CD deferred to a later milestone)

---

## Architecture Overview

### High-Level System Diagram

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
│    └─────────────┘                                          │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Technology Stack

**Interfaces**:
- `typer`: CLI framework — type-hint-driven, auto-generates help docs
- `FastMCP`: MCP server — generic tool names, JSON schema parameters
- Python library: direct import for scripting/automation

**Core Engine**:
- `pydantic`: Data validation — typed models for configs, API payloads, responses
- `ruamel.yaml`: YAML parsing — preserves comments and formatting in tier templates
- Python 3.10+: match/case, type unions, modern syntax

**Adapters**:
- `httpx` (async): HTTP client — concurrent API calls across app keys, 60s timeout, gzip
- `python-dotenv`: Credential loading from `.env` file

**Infrastructure**:
- Local filesystem: `.admedi/` directory for configs, sync logs, snapshots
- No database, no cloud services, no external dependencies beyond the LevelPlay API

### Cost Structure

- **Infrastructure cost**: $0/mo — runs entirely on developer's machine
- **LevelPlay API**: Free (included with LevelPlay account)
- **Dependencies**: All open-source Python packages
- **Net cost for Mochibits**: $0/mo additional

---

## Core Components Design

### 1. LevelPlay Adapter

**Purpose**: Implements the MediationAdapter interface for Unity LevelPlay. Handles all API communication including OAuth authentication, request construction, response parsing, rate limit handling, and error recovery.

**Features**:
- OAuth 2.0 Bearer token management (secretKey + refreshToken → JWT, 60-min expiry, auto-refresh before expiry)
- All read endpoints: `list_apps()`, `get_groups()`, `get_instances()`, `get_placements()`, `get_reporting()`
- All write endpoints: `create_group()`, `update_group()`, `delete_group()`, `create_instances()`, `update_instances()`, `delete_instance()`
- Rate limit awareness with exponential backoff (Groups: 4K/30min, Instances: 8K/30min, Reporting: 8K/hr)
- Both Groups API v2 (legacy) and v4 (current) support

**System Flow**:
```
authenticate()
  │ GET /partners/publisher/auth (secretKey + refreshToken in headers)
  ↓
JWT returned (60-min expiry)
  │ cached, auto-refreshes when exp claim < 5min remaining
  ↓
get_groups(app_key)
  │ GET /levelPlay/groups/v4/{appKey}
  ↓
Raw JSON response
  │ parse into normalized Group + WaterfallConfig models
  ↓
Normalized internal models returned to ConfigEngine
```

**Technical Design**:
- Async-first: all methods are `async def`, called via `httpx.AsyncClient`
- Concurrent app queries: when syncing 18 surfaces, fire all GET requests concurrently (respecting rate limits)
- Atomic per-app writes: if a write fails on one app, other apps are unaffected
- Never log credentials: auth tokens are masked in all log output

**Integration Points**:
- LevelPlay REST API (`platform.ironsrc.com`): All CRUD operations
- ConfigEngine: Provides normalized models for diffing and receives write instructions from Applier

### 2. ConfigEngine

**Purpose**: Orchestrates the full config-as-code cycle: load a YAML tier template, diff it against live remote configs, and apply changes. This is the brain of Admedi — it coordinates between the local config, the mediation adapter, and the storage adapter.

**Features**:
- YAML template loading and validation (via Loader)
- Per-app diffing against live remote config (via Differ)
- Batch diff across entire portfolio with preview table
- Confirmed apply with per-app atomic operations (via Applier)
- Sync log recording and config snapshot capture (via Storage Adapter)

**System Flow**:
```
User runs `admedi sync-tiers`
  ↓
Loader reads YAML template → normalized TierTemplate
  ↓
For each app_key in portfolio:
  │ MediationAdapter.get_groups(app_key)
  │ Normalize remote config → Group models
  │ Differ.compare(local_template, remote_groups)
  │ → DiffReport (added countries, removed, floor price changes)
  ↓
All DiffReports collected → preview table printed
  ↓
User confirms (y/n)
  ↓
For each app_key with changes:
  │ Applier calls MediationAdapter.update_group() / create_group()
  │ StorageAdapter.save_sync_log()
  │ StorageAdapter.save_snapshot()
  ↓
Summary: N apps updated, M groups modified, K errors
```

**Technical Design**:
- Loader validates YAML against pydantic schema — catches malformed configs before any API calls
- Differ is a pure function: no side effects, fully testable with mock data
- Applier wraps each app in a try/except — partial portfolio failure is reported, not fatal
- DiffReport is a structured object: `additions[]`, `removals[]`, `modifications[]` per group per app

**Integration Points**:
- Loader ← YAML file on disk
- Differ ← Loader output + MediationAdapter output
- Applier → MediationAdapter write methods
- Storage → sync logs and snapshots after each apply

### 3. YAML Tier Template

**Purpose**: The source of truth for ad mediation configuration. Defines country tiers, floor prices, and portfolio scope in a human-readable, version-controllable format. This is what the user creates and maintains — everything else derives from it.

**Features**:
- Tier definitions with country lists and optional floor prices
- Portfolio scope: which app_keys to manage
- Ad format targeting: which formats (banner, interstitial, rewarded) each tier applies to
- Readable by non-engineers — a monetization manager should understand the YAML without explanation

**Template Structure**:
```
# admedi.yaml — Shelf Sort tier template
mediator: levelplay
portfolio:
  - app_key: "abc123"
    name: "Shelf Sort Android"
    platform: android
  - app_key: "def456"
    name: "Shelf Sort iOS"
    platform: ios
  # ... all 6 apps × 3 platforms

tiers:
  tier_1:
    name: "Tier 1 — US Only"
    countries: [US]
    position: 1

  tier_2:
    name: "Tier 2 — Premium Markets"
    countries: [AU, CA, DE, GB, JP, NZ, KR, TW]
    position: 2

  tier_3:
    name: "Tier 3 — Mid Markets"
    countries: [FR, NL]
    position: 3

  catch_all:
    name: "All Countries"
    countries: []  # empty = all remaining
    position: 4

ad_formats: [banner, interstitial, rewarded_video]
```

**Technical Design**:
- Parsed by `ruamel.yaml` to preserve comments and formatting on round-trip
- Validated by pydantic: country codes checked against ISO 3166-1 alpha-2, no duplicate countries across tiers
- Empty `countries: []` on catch-all means "all countries not explicitly assigned to another tier"
- `ad_formats` controls which formats get the tier treatment — allows some formats to be excluded

**Integration Points**:
- ConfigEngine.Loader reads this file
- ConfigEngine.Differ compares this against remote Group configs

### 4. CLI (typer)

**Purpose**: Terminal interface for all Admedi operations. The primary way Mochibits will interact with the tool day-to-day. Commands map 1:1 to ConfigEngine operations.

**Features**:
- `admedi sync-tiers` — load template, diff against remote, preview changes, apply on confirm
- `admedi audit` — read-only diff, report mismatches without applying changes
- `admedi revenue` — pull reporting data, show per-tier eCPM averages, flag underperformers
- `admedi manage-instances` — bulk instance operations (add, remove, enable, disable)
- `admedi status` — show current portfolio overview (apps, platforms, last sync time)
- Rich terminal output: colored diffs, formatted tables, progress bars for multi-app operations

**User Flow**:
```
$ admedi audit
Loading template from ./admedi.yaml...
Pulling configs for 18 surfaces...

┌────────────────────────┬──────────┬─────────────────────────┐
│ App                    │ Status   │ Issues                  │
├────────────────────────┼──────────┼─────────────────────────┤
│ Shelf Sort Android     │ ⚠ DRIFT │ Tier 2 missing KR       │
│ Shelf Sort iOS         │ ✅ OK   │                         │
│ Shelf Sort Amazon      │ ⚠ DRIFT │ Tier 3 has extra MY     │
│ ...                    │          │                         │
└────────────────────────┴──────────┴─────────────────────────┘

3 apps have config drift. Run `admedi sync-tiers` to fix.
```

**Technical Design**:
- typer handles argument parsing, help generation, and command routing
- Each command is a thin wrapper that calls ConfigEngine methods
- `--dry-run` flag on all write commands for safe testing
- `--app` flag to target a single app instead of full portfolio
- `--format json` option for scriptable output
- Exit codes: 0 = success, 1 = drift detected (audit), 2 = error

**Integration Points**:
- ConfigEngine: all commands delegate to engine methods
- `.env` file: credentials loaded at startup via `python-dotenv`
- `.admedi/` directory: sync logs and snapshots stored here

### 5. FastMCP Server

**Purpose**: Expose Admedi as MCP tools for AI agents. Enables Claude Code, Cursor, and other MCP-compatible tools to manage ad mediation configs conversationally. Generic tool names — the active mediator is determined by config, not tool name.

**Features**:
- `get_groups(app_key)` — pull current mediation groups for an app
- `sync_tiers(template_path, app_keys?)` — run the full sync cycle with confirmation
- `audit(template_path, app_keys?)` — read-only drift detection
- `revenue_check(app_keys, days?)` — pull reporting data and flag underperformers
- `manage_instances(action, network, app_keys?)` — bulk instance operations
- `status()` — portfolio overview

**System Flow**:
```
Claude Code user: "check if my configs are in sync"
  ↓
MCP tool call: audit(template_path="./admedi.yaml")
  ↓
FastMCP Server receives call
  ↓
Delegates to ConfigEngine.audit()
  ↓
Returns structured JSON: { apps: [...], drift_count: 3, details: [...] }
  ↓
Claude Code formats response for user
```

**Technical Design**:
- FastMCP server loads credentials from `.env` at startup — no credentials flow through MCP protocol
- Tool names are generic (`get_groups`, not `levelplay_get_groups`)
- JSON schema parameters for all tools — MCP clients get auto-generated descriptions
- Stateless: server can crash and restart with no data loss (all state in storage adapter)
- Confirmation flow for write operations: tool returns preview, requires second call to confirm

**Integration Points**:
- ConfigEngine: all tools delegate to engine methods (same as CLI)
- `.env` file: credentials loaded once at server startup
- MCP protocol: standard tool definitions, JSON schema parameters and responses

### 6. Local File Storage Adapter

**Purpose**: Zero-dependency persistence for the open-source default. Stores config templates, sync logs, and config snapshots as flat files in a `.admedi/` directory. No database needed.

**Features**:
- `save_config(config)` — write YAML config to `.admedi/configs/`
- `load_config(config_id)` — read config from disk
- `save_sync_log(log)` — append JSON log entry to `.admedi/logs/`
- `list_sync_history(app_key)` — read and filter log entries
- `save_snapshot(app_key, remote_config)` — store full remote config JSON in `.admedi/snapshots/`

**Technical Design**:
- Sync logs are append-only JSON lines files (one line per sync operation) — easy to grep and parse
- Snapshots are stored as `{app_key}_{timestamp}.json` — full remote config capture for audit
- Config files are YAML — human-readable, git-diffable
- No locking or concurrency handling needed — single-user tool at this stage
- Directory structure:
  ```
  .admedi/
  ├── configs/          # YAML tier templates
  ├── logs/             # JSON lines sync logs
  └── snapshots/        # Point-in-time remote config captures
  ```

**Integration Points**:
- ConfigEngine: called after every sync operation to persist logs and snapshots
- CLI `status` command: reads sync history to show last sync time per app

---

## Implementation Phases

### Phase 1: Foundation

**Objective**: Set up the project structure and build the LevelPlay adapter with full API coverage.

**Deliverables**:
- Repo scaffolding: `pyproject.toml`, package structure (`admedi/adapters/`, `admedi/engine/`, `admedi/cli/`, `admedi/mcp/`, `admedi/storage/`, `admedi/models/`)
- Pydantic models for all 9 core entities: App, TierTemplate, Group, WaterfallConfig, Instance, Placement, SyncLog, ConfigSnapshot, Credential
- MediationAdapter abstract interface with 12 method signatures
- StorageAdapter abstract interface with 5 method signatures
- LevelPlay adapter: OAuth 2.0 auth flow (authenticate, token caching, auto-refresh)
- LevelPlay adapter: all read endpoints (`list_apps`, `get_groups`, `get_instances`, `get_placements`, `get_reporting`)
- LevelPlay adapter: all write endpoints (`create_group`, `update_group`, `delete_group`, `create_instances`, `update_instances`, `delete_instance`)

**Configuration**:
- `.env` file with `LEVELPLAY_SECRET_KEY` and `LEVELPLAY_REFRESH_TOKEN`
- `httpx.AsyncClient` with 60s timeout, gzip support, retry on 429/500

**Success Criteria**:
- ✅ OAuth token obtained and auto-refreshes before expiry
- ✅ `list_apps()` returns all Shelf Sort apps correctly
- ✅ `get_groups()` returns normalized Group models matching what's in the LevelPlay dashboard
- ✅ Write operations tested on a single non-production app without corrupting config

### Phase 2: ConfigEngine

**Objective**: Build the core config-as-code pipeline: load YAML templates, diff against remote, apply changes.

**Deliverables**:
- YAML tier template format specification and example Shelf Sort template
- Loader: parse YAML → validated TierTemplate via pydantic, reject invalid configs with clear error messages
- Differ: compare TierTemplate against remote Groups per app, produce structured DiffReport
- Applier: execute DiffReport by calling adapter write methods, record results
- DiffReport model: additions, removals, modifications per group per app with human-readable descriptions

**Testing**:
- Differ tested with mock data: known local template vs known remote config → expected DiffReport
- Applier tested in dry-run mode first (preview only, no writes)
- End-to-end: load Shelf Sort template → diff against one real app → verify DiffReport accuracy manually

**Success Criteria**:
- ✅ Loader parses Shelf Sort YAML template without errors
- ✅ Differ correctly identifies that South Korea is missing from Tier 2 on apps where it hasn't been added
- ✅ Differ correctly identifies Malaysia needs removal from Tier 3
- ✅ Applier successfully updates one test app's tier config via API

### Phase 3: Interfaces

**Objective**: Build the CLI and MCP server, plus the local file storage adapter.

**Deliverables**:
- CLI commands: `sync-tiers`, `audit`, `revenue`, `manage-instances`, `status`
- CLI flags: `--dry-run`, `--app`, `--format json`, `--config` (template path)
- MCP server: 6 tools (`get_groups`, `sync_tiers`, `audit`, `revenue_check`, `manage_instances`, `status`)
- Local file storage adapter: save/load configs, sync logs, snapshots in `.admedi/`
- Rich terminal output: colored diffs, tables, progress indicators

**Success Criteria**:
- ✅ `admedi audit` runs against full portfolio and produces accurate drift report
- ✅ `admedi sync-tiers --dry-run` shows preview without making changes
- ✅ MCP tools callable from Claude Code with correct JSON schema
- ✅ Sync logs persisted in `.admedi/logs/` after every operation
- ✅ Config snapshots captured in `.admedi/snapshots/` after every sync

### Phase 4: Dogfood

**Objective**: Validate Admedi on the real Shelf Sort portfolio. Manage all 18 configuration surfaces exclusively via CLI and MCP for 2+ weeks.

**Deliverables**:
- Production Shelf Sort YAML tier template matching current dashboard config exactly
- Full portfolio audit — identify and fix all existing config drift
- First real sync: apply the template across all 18 surfaces
- 2-week dogfood period: all tier changes made via `admedi sync-tiers`, zero dashboard touches
- Bug fixes discovered during dogfood
- MCP dogfood: use Claude Code to query portfolio status and make changes conversationally

**Production Launch**:
- Shelf Sort portfolio fully under Admedi management
- Shelf Sort YAML template becomes the source of truth for tier config
- Any future tier changes go through template edit → `admedi sync-tiers` workflow

**Success Criteria**:
- ✅ All 18 surfaces synced from one template in < 2 minutes
- ✅ Zero config drift detected after sync (audit returns clean)
- ✅ 2+ weeks of daily use with zero manual dashboard configuration
- ✅ At least one real tier adjustment (country promotion/demotion) applied via CLI
- ✅ MCP tools used successfully via Claude Code for at least 3 different operations

---

## Success Metrics

### Functionality

**API Coverage**:
- Target: 100% of LevelPlay management endpoints (Groups v4, Instances v3, Placements v1, Reporting v1)
- Measured: Endpoint test coverage — each adapter method tested with real API call
- Why: Incomplete API coverage means falling back to the dashboard, which defeats the purpose

**Config Sync Accuracy**:
- Target: Zero drift after sync — `admedi audit` returns clean on all 18 surfaces
- Measured: Automated audit after every sync operation
- Why: If sync introduces errors or misses changes, the tool is worse than the dashboard

**Sync Speed**:
- Target: Full portfolio sync (18 surfaces) in < 2 minutes
- Measured: CLI timing output
- Why: 45 minutes of dashboard clicking → 2 minutes of CLI is the core value prop

### Reliability

**Auth Stability**:
- Target: Zero auth failures during 2-week dogfood (token auto-refresh works across multi-hour sessions)
- Measured: Error log monitoring during dogfood period
- Why: Auth failures mid-sync corrupt the operation and erode trust

**Rate Limit Handling**:
- Target: Zero hard 429 errors (backoff kicks in before hitting limits)
- Measured: Request log analysis
- Why: 18 concurrent surface queries could hit limits without proper throttling

### Usability

**Time to First Audit**:
- Target: < 10 minutes from install to first `admedi audit` output
- Measured: Timed walkthrough
- Why: If setup is painful, adoption dies. Quick time-to-value is critical.

---

## Testing Strategy

**Philosophy**: Production-grade quality from day one, but sized for the Shelf Sort portfolio. No CI/CD in this milestone — deferred to a later packaging phase.

### Test Coverage Approach
- **Unit Tests**: Differ logic (pure function, mock data), Loader validation (valid and invalid YAML inputs), pydantic model serialization/deserialization
- **Integration Tests**: LevelPlay adapter against real API (single test app, not full portfolio), ConfigEngine end-to-end with mock adapter
- **Manual Tests**: Full portfolio sync with visual verification against dashboard, MCP tools via Claude Code

### Quality Gates
- All unit tests pass before dogfood phase begins
- Integration test against one real app succeeds before full portfolio sync
- First full sync is `--dry-run` with manual DiffReport review before live apply

### What We're NOT Testing (Yet)
- Automated CI/CD pipeline (deferred to a later milestone)
- Mock adapter for CI (no real API calls in CI — deferred)
- Load testing or stress testing
- Multi-user concurrency (single-user tool)
- Cross-platform compatibility (runs on dev machine only)

---

## Key Outcomes

✅ **Shelf Sort portfolio fully managed via config-as-code**
   - All 18 surfaces synced from one YAML template
   - Zero manual dashboard configuration needed for tier changes

✅ **ConfigEngine pipeline proven on real production data**
   - Loader → Differ → Applier cycle works reliably
   - DiffReports are accurate and human-readable

✅ **LevelPlay adapter battle-tested**
   - OAuth auth, rate limits, error handling all proven against live API
   - Supports both legacy v2 and current v4 Groups API

✅ **MCP tools functional**
   - Admedi usable via Claude Code conversation
   - All 6 tools return structured, useful responses

✅ **YAML template format validated**
   - Template represents real Shelf Sort config accurately
   - Format is readable by non-engineers and editable in any text editor

---

## Why Internal-First?

**Eat Our Own Dogfood**:
- We have 18 real configuration surfaces that need this tool today
- Building for ourselves means we catch real issues before anyone else hits them
- Shelf Sort portfolio is the perfect test case — enough complexity to validate, small enough to debug

**Zero Adoption Risk**:
- No need to convince anyone to try an unproven tool — we are the customer
- No marketing, no onboarding flows, no support requests
- Focus 100% on the engine and interfaces

**Fast Iteration**:
- No backward compatibility concerns — we can change the YAML format daily
- No user communications needed for breaking changes
- Move fast, break things, fix them before going public

**Validates the Core Thesis**:
- If config-as-code doesn't save us time on 18 surfaces, it won't save anyone time
- If the Differ produces inaccurate diffs, the whole product premise is broken
- Prove it works before investing in open-source packaging

---

## Design Decisions & Rationale

### Why YAML over JSON for Config Templates?

- **Human-readable**: Monetization managers can read and edit YAML without developer help
- **Comments**: YAML supports inline comments — critical for documenting why a country is in a specific tier
- **Git-diffable**: YAML diffs are clean and meaningful in PRs
- **Established pattern**: Kubernetes, Docker Compose, GitHub Actions all use YAML for config-as-code

**Alternative Considered**: JSON (rejected — no comments, less readable, worse diffs)

### Why ruamel.yaml over PyYAML?

- **Round-trip preservation**: ruamel.yaml preserves comments, formatting, and ordering when reading and writing YAML
- **Modern API**: Better error messages, safer defaults (no arbitrary code execution)
- **Active maintenance**: Regular updates, good Python 3.10+ support

**Alternative Considered**: PyYAML (rejected — loses comments on round-trip, security concerns with `yaml.load`)

### Why Async-First HTTP Client?

- **Concurrent multi-app queries**: 18 surfaces queried concurrently instead of sequentially
- **Performance**: Sequential at 500ms/call = 9 seconds. Concurrent = ~1-2 seconds.
- **Proven pattern**: The abandoned ironSource lib already uses `httpx.AsyncClient` successfully

**Alternative Considered**: Synchronous `requests` (rejected — too slow for multi-app operations)

### Why Local File Storage as Default?

- **Zero dependencies**: No database to install, configure, or maintain
- **Git-friendly**: Config files and logs can be committed to the repo
- **Inspectable**: Any text editor can read sync logs and snapshots
- **Sufficient for single-user**: No concurrency concerns for Mochibits internal use

**Alternative Considered**: SQLite from day one (rejected — adds complexity without benefit for single-user internal tool)

---

## Risks & Mitigation

### Risk: LevelPlay API Behavior Differs from Documentation

**Impact**: High — adapter produces incorrect results or fails silently
**Probability**: Medium — Unity's documentation has historically had gaps
**Mitigation**:
- Test every endpoint against real API before trusting documentation
- Compare API responses against dashboard UI to verify correctness
- Log raw API responses during dogfood for debugging
- Fallback: adjust adapter to match actual API behavior, document discrepancies

### Risk: YAML Template Can't Represent Real Config Complexity

**Impact**: High — the core product concept fails
**Probability**: Low — we researched the LevelPlay config model thoroughly
**Mitigation**:
- Start by exporting current Shelf Sort config and building YAML from it (bottom-up)
- Design template format iteratively — adjust as edge cases emerge during dogfood
- Accept that template covers 80% case; complex per-app overrides can use `--app` flag

### Risk: Rate Limits Hit During Full Portfolio Sync

**Impact**: Medium — sync takes longer or partially fails
**Probability**: Low — 18 surfaces is well within the 4K/30min limit
**Mitigation**:
- Implement request counting and proactive throttling (not just reactive backoff)
- Batch where APIs support it (Instances API supports bulk operations)
- Log rate limit headers on every response to monitor headroom

### Risk: Active A/B Test Blocks API Calls

**Impact**: Medium — sync and audit fail for affected apps with no API workaround
**Probability**: Medium — A/B tests are common during monetization optimization
**Mitigation**:
- Detect A/B test status from Groups API response (`abTest` field) before attempting writes
- Skip affected apps with clear warning: "App X has active A/B test — skipping (end test in dashboard to sync)"
- Document this as a known LevelPlay API limitation

### Risk: Write Operation Corrupts Live Config

**Impact**: High — production ad mediation config broken
**Probability**: Low — atomic per-app operations with dry-run first
**Mitigation**:
- Always capture config snapshot before applying changes
- `--dry-run` is the default for first-time users
- Applier verifies the write succeeded by re-reading the config after apply
- Rollback plan: re-apply previous snapshot via the adapter

---

## Open Questions

### Template Design

- **How to handle per-app overrides?**: Some apps may need a country in a different tier than the template default. Options: override block in YAML, separate per-app template, or `--exclude-app` flag.
  - Decision: Start with a single global template. Add override syntax only if needed during dogfood.

- **Should floor prices be in the template?**: Floor prices are tier-specific in LevelPlay. Including them in YAML adds config power but also complexity.
  - Decision: Include optional `floor_price` per tier. Omitted = use remote default.

### API Behavior

- **Groups API v2 vs v4 — which to use for writes?**: v4 is documented as current but v2 may still be required for some operations.
  - Decision: Test both during Phase 1. Use v4 as primary, fall back to v2 if v4 doesn't support a needed operation.

- **Does `update_group` support partial updates?**: Or does it require sending the full group config?
  - Decision: Test during Phase 1. If full config required, always read-then-modify-then-write.

- **Instances API v1 vs v3 — which is current?**: The existing ironSource lib uses v3 (`/instances/v3`) but current LevelPlay docs show v1 (`/instances/v1`). Need to verify which version is active.
  - Decision: Test both during Phase 1, same approach as Groups v2 vs v4.

---

## Next Steps

**Immediate** (Start Here):
1. Create repo structure and `pyproject.toml`
2. Define pydantic models for all 9 core entities
3. Implement LevelPlay OAuth auth flow and test with real credentials

**After Foundation**:
1. Build read endpoints and verify against Shelf Sort dashboard
2. Design YAML template format by exporting current Shelf Sort config
3. Implement Differ with mock data tests

**Before Milestone Complete**:
1. Full portfolio sync with `--dry-run` reviewed manually
2. First live sync with snapshot backup
3. 2-week dogfood period with daily use
4. Document all bugs and edge cases found during dogfood

---

## Related Documents

- [Roadmap](./references/admedi-roadmap.md) — Full milestone roadmap
- [Architecture Doc](./references/admedi-architecture.md) — Complete technical architecture
- [Vision Doc](./references/admedi-vision.md) — Product vision and scope
- [API Reference](./references/levelplay-api-reference.md) — LevelPlay API endpoint documentation

---

*Document Status*: Design Complete — Ready for Implementation
*Last Updated*: February 2026
