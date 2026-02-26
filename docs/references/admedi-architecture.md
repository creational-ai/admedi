# Admedi — Architecture

**Version:** 1.0
**Repo:** `creational-ai/admedi`
**Input:** `admedi-vision.md` v1.7

## Overview

Admedi is a config-driven ad mediation management tool with two adapter boundaries: mediation platform adapters (how we talk to LevelPlay, MAX, etc.) and storage adapters (how we persist state). The core engine reads a YAML config template, diffs it against live mediation configs pulled via platform APIs, and applies changes across an entire app portfolio. Four interfaces expose this engine: Python library, CLI, FastMCP server, and Cowork plugin.

## Architecture

### System Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                      INTERFACE LAYER                        │
│                                                             │
│   ┌─────────┐  ┌─────────┐  ┌──────────┐  ┌────────────┐  │
│   │ Python  │  │   CLI   │  │ FastMCP  │  │  Cowork    │  │
│   │ Library │  │ (typer) │  │  Server  │  │  Plugin    │  │
│   └────┬────┘  └────┬────┘  └────┬─────┘  └─────┬──────┘  │
│        │            │            │               │          │
│        └────────────┴──────┬─────┴───────────────┘          │
│                            │                                │
├────────────────────────────┼────────────────────────────────┤
│                     CORE ENGINE                             │
│                            │                                │
│   ┌────────────────────────▼─────────────────────────────┐  │
│   │              ConfigEngine                            │  │
│   │  ┌──────────┐  ┌──────────┐  ┌───────────────────┐  │  │
│   │  │  Loader  │  │  Differ  │  │    Applier        │  │  │
│   │  │ (YAML)   │  │ (diff)   │  │ (sync to remote)  │  │  │
│   │  └──────────┘  └──────────┘  └───────────────────┘  │  │
│   └──────────────────────────────────────────────────────┘  │
│                            │                                │
├────────────────────────────┼────────────────────────────────┤
│                     ADAPTER LAYER                           │
│              ┌─────────────┼─────────────┐                  │
│              │             │             │                  │
│   ┌──────────▼──┐  ┌──────▼──────┐  ┌──▼───────────────┐  │
│   │  Mediation  │  │  Storage    │  │  (future)        │  │
│   │  Adapters   │  │  Adapters   │  │  Notification    │  │
│   │             │  │             │  │  Adapters        │  │
│   │ ┌─────────┐ │  │ ┌─────────┐ │  └──────────────────┘  │
│   │ │LevelPlay│ │  │ │  Local  │ │                         │
│   │ │ (MVP)   │ │  │ │  File   │ │                         │
│   │ ├─────────┤ │  │ ├─────────┤ │                         │
│   │ │  MAX    │ │  │ │ SQLite  │ │                         │
│   │ │(future) │ │  │ ├─────────┤ │                         │
│   │ ├─────────┤ │  │ │Postgres │ │                         │
│   │ │ AdMob   │ │  │ │ (SaaS)  │ │                         │
│   │ │(future) │ │  │ └─────────┘ │                         │
│   │ └─────────┘ │  └─────────────┘                         │
│   └─────────────┘                                          │
│                                                             │
├─────────────────────────────────────────────────────────────┤
│                   EXTERNAL SERVICES                         │
│                                                             │
│   ┌─────────────┐  ┌──────────┐  ┌──────────────────────┐  │
│   │  LevelPlay  │  │  MAX     │  │  AdMob Mediation     │  │
│   │  REST API   │  │  API     │  │  API                 │  │
│   └─────────────┘  └──────────┘  └──────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

### Components

#### Interface Layer

##### Python Library
- **Purpose**: Programmatic access to all Admedi functionality
- **Inputs**: Method calls with typed arguments
- **Outputs**: Typed response objects
- **Dependencies**: Core Engine

##### CLI (typer)
- **Purpose**: Terminal-driven ad ops — sync tiers, audit configs, pull reports
- **Inputs**: CLI commands + flags + YAML config file path
- **Outputs**: Formatted terminal output (tables, diffs, confirmations)
- **Dependencies**: Core Engine, Python library

##### FastMCP Server
- **Purpose**: Expose Admedi as MCP tools for AI agents (Claude Code, Cursor, etc.)
- **Inputs**: MCP tool calls with JSON parameters
- **Outputs**: MCP tool responses (JSON)
- **Dependencies**: Core Engine, Python library
- **Note**: Tool names are generic (`get_groups`, `sync_tiers`) — no mediator prefix. Active mediator is determined by top-level config.

##### Cowork Plugin
- **Purpose**: Conversational AI interface for non-technical monetization managers
- **Inputs**: Natural language via Claude Desktop
- **Outputs**: Guided workflows with previews, diffs, confirmations
- **Dependencies**: FastMCP Server, Skills (SKILL.md files)

#### Core Engine

##### ConfigEngine
- **Purpose**: Orchestrates the load → diff → apply cycle across an entire app portfolio
- **Inputs**: YAML config template, list of target app keys, active mediation adapter
- **Outputs**: Diff reports, apply results, audit logs
- **Dependencies**: Mediation Adapter (for remote state), Storage Adapter (for persistence)

##### Loader
- **Purpose**: Parses YAML config templates into normalized internal data structures
- **Inputs**: YAML file path or string
- **Outputs**: Normalized `TierTemplate`, `WaterfallConfig`, `InstanceConfig` objects
- **Dependencies**: None (pure parsing)

##### Differ
- **Purpose**: Compares local config template against live remote config per app/platform/ad format
- **Inputs**: Normalized local config + normalized remote config
- **Outputs**: `DiffReport` — list of changes per app (additions, removals, modifications)
- **Dependencies**: None (pure comparison)

##### Applier
- **Purpose**: Executes the diff by calling mediation adapter write methods
- **Inputs**: `DiffReport`, target app keys, mediation adapter
- **Outputs**: `ApplyResult` — success/failure per app, changes made
- **Dependencies**: Mediation Adapter

#### Adapter Layer

##### Mediation Adapter Interface
- **Purpose**: Abstract interface that all mediation platform adapters implement
- **Inputs/Outputs**: Normalized data structures (not platform-specific JSON)
- **Dependencies**: Platform-specific REST API credentials

The interface defines these capabilities:
- `authenticate()` — obtain/refresh API token
- `list_apps()` — enumerate registered apps
- `get_groups(app_key)` — pull mediation groups (country tiers, waterfall)
- `create_group(app_key, group_config)` — create a new group
- `update_group(app_key, group_id, group_config)` — modify existing group
- `delete_group(app_key, group_id)` — remove a group
- `get_instances(app_key)` — list ad network instances
- `create_instances(app_key, instances)` — add instances
- `update_instances(app_key, instances)` — modify instances
- `delete_instance(app_key, instance_id)` — remove instance
- `get_placements(app_key)` — list placements
- `get_reporting(app_key, date_range, breakdowns)` — pull performance data

Each adapter translates between Admedi's normalized models and the platform's specific API format.

##### LevelPlay Adapter (MVP)
- **Purpose**: Implements the Mediation Adapter Interface for Unity LevelPlay
- **Dependencies**: LevelPlay REST API (`platform.ironsrc.com`)
- **Auth**: OAuth 2.0 Bearer (secretKey + refreshToken → JWT, 60-min expiry, auto-refresh)
- **Rate limits**: Groups API 4K/30min, Instances 8K/30min, Reporting 8K/hr
- **Endpoints**: Groups API v4, Mediation Management v2, Instances v3, Placements v1, Reporting v1, Application v6, ILR v3
- **Note**: Supports both v2 (legacy) and v4 (current) Groups API for backward compatibility

##### Storage Adapter Interface
- **Purpose**: Abstract interface for persisting config state, audit logs, and sync history
- **Capabilities**:
  - `save_config(config)` — persist a config template
  - `load_config(config_id)` — retrieve a config template
  - `save_sync_log(log)` — record what changed, when, on which apps
  - `list_sync_history(app_key)` — retrieve audit trail
  - `save_snapshot(app_key, remote_config)` — store point-in-time snapshot of remote state

##### Local File Adapter (open-source default)
- **Purpose**: Zero-dependency persistence using local YAML/JSON files
- **Storage**: Config templates as YAML files, sync logs as JSON in a `.admedi/` directory
- **Dependencies**: None (filesystem only)

##### SQLite Adapter (self-hosted option)
- **Purpose**: Lightweight relational persistence for users who want query capability without Postgres
- **Dependencies**: Python stdlib `sqlite3`

##### Postgres Adapter (SaaS)
- **Purpose**: Full relational persistence on Creational.ai's RDS
- **Dependencies**: `asyncpg`, existing RDS Postgres on AWS
- **Features**: Multi-tenant isolation, full audit history, config versioning

## Data Model

### Core Entities

| Entity | Purpose | Key Fields |
|--------|---------|------------|
| `App` | Registered app in a mediation platform | `app_key`, `name`, `platform` (Android/iOS/Amazon), `mediator` (levelplay/max/admob), `bundle_id` |
| `TierTemplate` | Reusable country tier definition | `name`, `tiers[]` (each with `name`, `countries[]`, `position`, `floor_price`) |
| `Group` | Normalized mediation group (maps to LevelPlay group, MAX ad unit group, etc.) | `group_id`, `name`, `countries[]`, `position`, `ad_format`, `floor_price`, `waterfall` |
| `WaterfallConfig` | Ordered list of ad sources within a group | `bidding_instances[]`, `tiers[]` (each with `tier_type`, `instances[]`) |
| `Instance` | An ad network instance within a waterfall | `instance_id`, `name`, `network`, `is_bidder`, `rate`, `countries_rate{}` |
| `Placement` | In-app ad placement with capping/pacing | `placement_id`, `name`, `ad_unit`, `ad_delivery`, `capping`, `pacing` |
| `SyncLog` | Record of a config sync operation | `timestamp`, `app_key`, `action`, `diff_summary`, `result` (success/failure), `user` |
| `ConfigSnapshot` | Point-in-time capture of remote mediation state | `app_key`, `timestamp`, `raw_config` (full JSON from platform API) |
| `Credential` | Platform API credentials | `mediator`, `secret_key`, `refresh_token`, `token_expiry` |

### Relationships

An `App` belongs to one `mediator` type. A `TierTemplate` applies to many `Apps` (one-to-many fan-out — the core sync operation). A `Group` belongs to one `App` and contains one `WaterfallConfig`. A `WaterfallConfig` contains many `Instances`. A `SyncLog` references one `App` and one `TierTemplate`. A `ConfigSnapshot` captures the full state of one `App` at a point in time (for audit/rollback).

**Note**: Keep it simple for first 200 users. The Local File adapter doesn't need to model relationships — just flat YAML files. SQLite/Postgres adapters add relational integrity.

## Data Flow

### Flow 1: Sync Tiers Across Portfolio

```
1. User runs `admedi sync-tiers` (or MCP tool call, or plugin skill)
2. ConfigEngine.Loader reads YAML tier template from config file
3. ConfigEngine iterates over all app_keys in portfolio config
4. For each app_key:
   a. MediationAdapter.get_groups(app_key) → pulls live remote config
   b. ConfigEngine normalizes remote config into internal Group models
   c. ConfigEngine.Differ compares local TierTemplate vs remote Groups
   d. Differ produces a DiffReport (added countries, removed countries,
      changed floor prices, new groups needed, etc.)
5. All DiffReports collected and presented to user as a preview table
6. User confirms (CLI prompt / MCP confirmation / plugin skill confirmation)
7. For each app_key with changes:
   a. ConfigEngine.Applier calls MediationAdapter.update_group() or
      create_group() per diff
   b. StorageAdapter.save_sync_log() records what changed
   c. StorageAdapter.save_snapshot() captures post-sync state
8. Summary returned: N apps updated, M groups modified, K errors
```

### Flow 2: Audit Config Consistency

```
1. User runs `admedi audit` (or plugin skill "check my configs")
2. ConfigEngine.Loader reads YAML tier template
3. For each app_key: MediationAdapter.get_groups(app_key)
4. ConfigEngine.Differ compares each app's live config against template
5. Differ flags inconsistencies:
   - App X Tier 2 is missing South Korea
   - App Y has a floor price mismatch on Tier 1
   - App Z has an extra group not in template
6. Audit report returned (no changes applied — read-only operation)
```

### Flow 3: Revenue Check

```
1. User runs `admedi revenue` or asks "how are my tiers performing"
2. ConfigEngine calls MediationAdapter.get_reporting() for each app_key
   with breakdowns by country, ad_format, mediation_group
3. Engine aggregates data across apps, computes per-tier eCPM averages
4. Flags underperforming countries (eCPM significantly below tier average)
5. Suggests tier adjustments (e.g., "Malaysia eCPM is $1.20 — below
   Tier 3 average of $3.40, consider moving to All Countries")
6. Report returned — no changes applied unless user explicitly requests
```

## Technology Stack

| Layer | Technology | Rationale |
|-------|------------|-----------|
| Language | Python 3.10+ | Match existing Creational.ai stack (Video Professor, Mission Control). Modern syntax (match/case, type unions). |
| HTTP Client | `httpx` (async) | Already proven in the ironSource lib. Async-first for concurrent multi-app API calls. |
| CLI Framework | `typer` | Modern, type-hint-driven CLI. Less boilerplate than Click. Auto-generates help docs. |
| MCP Server | `FastMCP` | Creational.ai's standard MCP framework. Already used in Mission Control and Video Professor. |
| Config Format | YAML (`ruamel.yaml`) | Human-readable, version-controllable. `ruamel.yaml` for round-trip comment/formatting preservation. |
| Data Validation | `pydantic` | Typed models for config parsing, API payloads, and response validation. Catches errors early. |
| Local Storage | YAML/JSON files | Zero-dependency default for open-source users. |
| SQLite | Python stdlib `sqlite3` | Self-hosted option, no external DB needed. |
| Postgres | `asyncpg` | SaaS storage — async, fast, matches existing RDS on Creational.ai AWS. |
| Testing | `pytest` + `pytest-asyncio` | Standard Python testing. Async support for adapter tests. |
| SaaS Hosting | AWS App Runner + RDS | Existing Creational.ai infra pattern. No new services to provision. |

## Integration Points

### LevelPlay REST API
- **Type**: REST API (HTTPS)
- **Purpose**: Read and write mediation configs (groups, instances, placements, reporting)
- **Contract**: OAuth 2.0 Bearer auth. Base URL `platform.ironsrc.com`. See `levelplay-api-reference.md` for full endpoint documentation.
- **Fallback**: If API is unreachable, fail gracefully with clear error. Never apply partial changes — atomic per-app operations. Cache last-known config via snapshots for audit comparison.

### FastMCP Protocol
- **Type**: MCP (Model Context Protocol)
- **Purpose**: Expose Admedi tools to AI agents
- **Contract**: Standard MCP tool definitions with JSON schema parameters. Generic tool names (`get_groups`, `sync_tiers`, `audit`, `revenue_check`).
- **Fallback**: MCP server is stateless — if it crashes, restart with no data loss (state is in storage adapter).

### Creational.ai RDS (SaaS only)
- **Type**: PostgreSQL via `asyncpg`
- **Purpose**: Persistent storage for SaaS multi-tenant deployments
- **Contract**: Postgres adapter implements StorageAdapter interface. Connection string via environment variable.
- **Fallback**: If DB is unreachable, SaaS operations fail with clear error. CLI/open-source users unaffected (they use local storage).

## Security Considerations

**Philosophy**: Production-grade from day one, but sized for first 200 users. Dedicated security milestone comes after product-market fit.

### Authentication & Authorization
- **Open-source**: Credentials stored in `.env` file (standard `python-dotenv`). User is responsible for file permissions. Single-user model — no auth layer needed.
- **SaaS**: Multi-tenant auth via Creational.ai's existing auth infrastructure. API keys per tenant. Credentials encrypted at rest in RDS.
- **MCP**: Credentials loaded from `.env` at server startup. No credentials transmitted over MCP protocol — the MCP server holds them server-side.

### Data Protection
- **Sensitive data**: Mediation platform API keys (`secretKey`, `refreshToken`). These grant full read/write access to a studio's ad config.
- **At rest**: Open-source — `.env` file, user-managed. SaaS — encrypted columns in Postgres (AES-256 via AWS RDS encryption).
- **In transit**: All API calls over HTTPS. No plaintext credential transmission.
- **Logging**: Credentials NEVER logged. Sync logs record actions and results, not auth tokens.

### Known Risks (Acceptable for MVP)
- Single `.env` file holds all credentials — acceptable for single-user CLI usage, must be hardened for SaaS
- No role-based access control in open-source version — single-user tool, so not needed yet
- Rate limit handling is basic (exponential backoff) — sufficient for MVP portfolio sizes

## Observability

**Philosophy**: Just enough to debug issues and understand usage. Expand when scaling.

### Logging
- Python `logging` module with structured JSON output
- Log levels: `INFO` for sync operations, `WARNING` for rate limits / retries, `ERROR` for API failures
- Logs go to stdout (CLI) or CloudWatch (SaaS via App Runner)

### Monitoring
- **Health**: SaaS endpoint `/health` — checks DB connectivity and API reachability
- **Key metrics**: Sync success/failure rate, API response times, rate limit hits
- **Alerting**: CloudWatch alarms on error rate spikes (SaaS only). Open-source users see errors in CLI output.

### Analytics
- **Usage**: Sync frequency, number of apps managed, most-used commands/tools
- **Open-source**: Optional anonymous telemetry (opt-in, clearly documented). Disabled by default.
- **SaaS**: Full usage tracking per tenant for billing and product analytics

## Key Design Decisions

### Decision 1: Dual Adapter Boundaries (Mediation + Storage)
- **Context**: Need to support multiple mediation platforms AND multiple storage backends
- **Options Considered**: Single monolithic class per platform; microservices per adapter; adapter interface pattern
- **Decision**: Two abstract interfaces — `MediationAdapter` and `StorageAdapter` — each with swappable implementations
- **Rationale**: Same OOP pattern, proven in countless infrastructure tools (Terraform providers, database drivers). Adding a new mediator or storage backend is implementing an interface, not modifying core logic. Keeps core engine testable with mock adapters.

### Decision 2: YAML Config as Source of Truth
- **Context**: Need a way to define "what tiers should look like" that's version-controllable and human-readable
- **Options Considered**: JSON config, database-stored config, GUI-driven config
- **Decision**: YAML file as the canonical tier template, stored alongside the codebase or in a config repo
- **Rationale**: Git-diffable, human-readable, portable. Studios can PR tier changes through their existing review process. JSON is less readable; DB-stored config requires a running service; GUI defeats the config-as-code purpose.

### Decision 3: Async-First HTTP Client
- **Context**: Syncing across 18+ app surfaces means many API calls
- **Options Considered**: Synchronous requests (simple), async with `httpx` (concurrent), threaded pool
- **Decision**: `httpx.AsyncClient` for all API calls
- **Rationale**: Already proven in the ironSource lib. Concurrent calls across app keys significantly reduces total sync time. A 18-surface sync with sequential calls at ~500ms each = 9 seconds. With async concurrency = ~1-2 seconds.

### Decision 4: FastMCP over Custom API
- **Context**: Need an AI-agent-compatible interface
- **Options Considered**: Custom REST API with OpenAPI spec, LangChain tools, FastMCP
- **Decision**: FastMCP server
- **Rationale**: Already the Creational.ai standard (Mission Control, Video Professor). Native Claude integration. MCP is becoming the industry standard for AI tool integration. A custom REST API would require each AI agent to build a custom integration.

### Decision 5: Generic MCP Tool Names
- **Context**: MCP tools could be named per-mediator (`levelplay_get_groups`) or generically (`get_groups`)
- **Options Considered**: Per-mediator prefixed names; generic names with mediator in config
- **Decision**: Generic names. Active mediator determined by top-level config, not tool name.
- **Rationale**: Tool interface stays stable as adapters are added. An AI agent's prompts and workflows don't need to change when switching from LevelPlay to MAX. The mediator is an infrastructure detail, not a user-facing concern.

### Decision 6: Install from GitHub, Not PyPI
- **Context**: Need a distribution mechanism for the Python package
- **Options Considered**: PyPI (traditional), GitHub direct install, private package registry
- **Decision**: `pip install git+https://github.com/creational-ai/admedi`
- **Rationale**: No PyPI name conflicts or overhead. Creational.ai controls the namespace. Users install from the single source of truth. Version pinning via git tags (`@v1.0.0`). For SaaS, package is deployed internally — no public registry needed.

## Constraints & Assumptions

- LevelPlay API remains stable and accessible (v2 and v4 endpoints). Unity has a history of API churn — we support both versions as a hedge.
- Single mediator per Admedi deployment at MVP. Multi-mediator (e.g., LevelPlay for some apps, MAX for others in the same config) is a post-MVP feature.
- Rate limits are sufficient for portfolios up to ~50 apps. Beyond that, need to implement smarter batching and request scheduling.
- The YAML config template covers the 80% case (country tiers, floor prices, waterfall priority). Advanced per-app customization (A/B tests, segments) remains dashboard-only.
- Async Python is required — the HTTP client, MCP server, and Postgres adapter all use async/await.

## Future Considerations

- **MAX adapter** — Research API availability, implement adapter if public REST API exists
- **AdMob adapter** — Same research needed
- **Multi-mediator config** — Single YAML that targets different mediators per app (e.g., apps 1-3 on LevelPlay, apps 4-6 on MAX)
- **Config rollback** — Use ConfigSnapshots to revert an app to a previous state
- **Scheduled syncs** — SaaS feature: cron-style automatic tier sync + drift detection
- **Notification adapter** — Third adapter boundary: Slack/Discord/email alerts on sync results or config drift
- **Dedicated security milestone** — RBAC, credential rotation, API key scoping after 200+ users
