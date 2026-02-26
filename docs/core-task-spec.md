# Core Task Spec

## Milestone Overview

Build and validate the core Admedi engine as an internal tool for Mochibits' Shelf Sort portfolio. Completing this milestone proves that config-as-code works for ad mediation — a single YAML template can drive tier sync across 18 configuration surfaces (6 apps x 3 platforms), replacing 45+ minutes of manual dashboard work with one CLI command. This unlocks the open-source, SaaS, and multi-mediator milestones.

## Project

[Admedi](./references/admedi-vision.md) — Config-as-code tool for ad mediation management.

## Task Dependency Diagram

```
┌──────────────────────────────────────┐
│  Task: Project Foundation            │
│  Scaffolding, models, interfaces     │
└──────────────────┬───────────────────┘
                   │
                   ▼
┌──────────────────────────────────────┐
│  Task: LevelPlay Authentication      │
│  OAuth auth, token mgmt, list_apps() │
└──────────────────┬───────────────────┘
                   │
                   ▼
┌──────────────────────────────────────┐
│  Task: LevelPlay Read & Write Ops    │
│  All 10 read & write endpoints +     │
│  rate limits + A/B test detection    │
└──────────────────┬───────────────────┘
                   │
                   ▼
┌──────────────────────────────────────┐
│  Task: ConfigEngine Pipeline         │
│  YAML template + Loader + Differ +   │
│  Applier + DiffReport                │
└──────────────────┬───────────────────┘
                   │
                   ▼
┌──────────────────────────────────────┐
│  Task: CLI, MCP Server & Storage     │
│  5 CLI commands, 6 MCP tools,        │
│  local file storage adapter          │
└──────────────────┬───────────────────┘
                   │
                   ▼
┌──────────────────────────────────────┐
│  Task: Portfolio Dogfood             │
│  Real Shelf Sort sync + 2-week       │
│  production validation               │
└──────────────────────────────────────┘
```

**Parallel tracks**: None — this is a sequential dependency chain. Each task builds directly on the previous. Authentication requires models; reads/writes require auth; the engine requires adapter operations; interfaces require the engine; dogfood requires interfaces. No meaningful work can be parallelized without introducing mocks that would need to be replaced.

**Note**: This is a plan - NO status indicators in diagram.

## Reference Repositories

**Full analysis**: [reference-repo-analysis.md](./references/reference-repo-analysis.md) — 8 repos analyzed in depth.

We are NOT forking any repo. We build from the ground up with our own architecture (pydantic models, adapter pattern, async-first). But these repos are the best source of truth for how APIs actually behave, what patterns work, and what anti-patterns to avoid. Study them before writing each task.

### Primary: ironSource/mobile-api-lib-python

**Repo**: [github.com/ironSource/mobile-api-lib-python](https://github.com/ironSource/mobile-api-lib-python) (Apache-2.0, abandoned Dec 2022)
**Local clone**: `references/ironsource-python-lib/`

The single best source of truth for LevelPlay API behavior — real request/response formats that Unity's docs often omit or get wrong.

| Source File | What It Teaches | Used In |
|-------------|-----------------|---------|
| `base_api.py` | Auth flow: JWT `exp` claim checking, auto-refresh, credential invalidation on change | Task: LevelPlay Authentication |
| `utils.py` | httpx patterns: 60s timeout, gzip, User-Agent, `ResponseInterface` normalization | Task: LevelPlay Authentication |
| `monetize_api/__init__.py` | Enums: Networks(33), AdUnits, Metrics(26), Breakdowns(15), Platform — exact API string values | Task: Project Foundation |
| `monetize_api/monetize_api.py` | 18 endpoint methods: URL paths, request body construction, response parsing | Task: LevelPlay Read & Write |
| `monetize_api/instance_config.py` | Per-network credential fields — what the API accepts for instance CRUD | Task: Project Foundation |
| `monetize_api/mediation_group_priority.py` | Waterfall structure: 3 tiers + bidders, TierType (manual/sortByCpm/optimized/bidding) | Task: Project Foundation |
| `monetize_api/placement_config.py` | Placement fields: capping (limit/interval), pacing (minutes), adDelivery (0/1) | Task: Project Foundation |
| URL constants in `monetize_api.py` | Confirmed working paths: `/instances/v3` (not v1), `/mediation/management/v2` for groups | Task: LevelPlay Read & Write |

### Cross-Reference: ironSource/mobile-api-lib-js

**Local clone**: `references/ironsource-js-lib/`

TypeScript type definitions cross-reference pydantic models. Has the most up-to-date network list (29 networks) and reveals API quirks not visible in the Python lib:
- Delete placement uses **request body** (not query params): `{appKey, adUnit, id}`
- Placement update **cannot rename** — API silently ignores name changes
- Instance appConfig: strip empty-value configs before sending
- Optimized tier type **cannot coexist** with bidders (business rule validated in code)
- Bid batch size limit: 9,998 items (undocumented)

### Patterns Adopted from Other Repos

| Pattern | Source Repo | Applied In |
|---------|------------|------------|
| Generic diff: `compute_diff(desired, current, matcher)` → `{additions, modifications, removals}` | SotiAds `listChanges()` | Task: ConfigEngine Pipeline |
| Default/override hierarchy in config | SotiAds YAML schema | Task: ConfigEngine Pipeline |
| Per-app error isolation (one failure doesn't abort portfolio) | SotiAds main loop | Task: LevelPlay Read & Write Operations, ConfigEngine Pipeline |
| Country fallback: try specific country, then fall back to ALL | OM-Server `Placement.java` | Task: ConfigEngine Pipeline |
| Coordinator singleton for MCP server | Google MCP GMS `coordinator.py` | Task: CLI, MCP Server & Storage |
| `ToolError` for user-facing MCP errors + `mask_error_details` | Google MCP GMS `tools/api.py` | Task: CLI, MCP Server & Storage |
| `output_schema` on MCP tool decorators | Google MCP GMS `tools/api.py` | Task: CLI, MCP Server & Storage |
| MCP Resources for reference data (passive endpoints) | Google MCP GMS `tools/docs.py` | Task: CLI, MCP Server & Storage |
| Fully mocked tests (no live credentials required) | Google MCP GMS `tests/` | All tasks except Portfolio Dogfood |
| Tagged per-app logging | SotiAds `consola.withTag()` | Task: Portfolio Dogfood |

### Anti-Patterns to Avoid

These are real problems found in the reference repos. Each has a concrete fix applied in our task designs.

| Anti-Pattern | Found In | Our Fix |
|-------------|----------|---------|
| **New httpx client per request** (no connection pooling) | ironSource Python `utils.py` | Persistent `httpx.AsyncClient` — one session per adapter instance |
| **Zero retry logic** (failures are final) | ironSource Python, SotiAds | Exponential backoff on 429/500, token refresh on 401 |
| **Zero rate limit awareness** | ironSource Python, SotiAds | Request counting + proactive throttling per endpoint |
| **42 classes for 33 networks** (1549 lines of boilerplate) | ironSource Python `instance_config.py` | Data-driven network schema registry — config as data, not code |
| **Generic `Exception` everywhere** | ironSource Python `monetize_api.py` | Typed exceptions: `AuthError`, `RateLimitError`, `ApiError`, `ValidationError` |
| **Manual JWT base64 splitting** | ironSource Python `base_api.py` | `PyJWT` library for proper JWT decoding |
| **`utcnow()` (deprecated Python 3.12+)** | ironSource Python `base_api.py` | Timezone-aware datetimes throughout |
| **No dry-run mode** (every sync is live) | SotiAds | Dry-run is the default — must explicitly confirm to apply |
| **No config snapshot before mutation** | SotiAds | Snapshot remote state before every write operation |
| **Not idempotent** (overwrites entire group) | SotiAds `syncMediationGroup()` | Diff-based minimal writes — re-running with no changes = zero API calls |
| **Schema defined but never validated** | SotiAds `read.ts` | Pydantic validates on model construction — invalid config fails fast |
| **Credentials loaded at import time** | Google MCP Official `utils.py` | Lazy initialization — credentials loaded on first use |
| **No audit trail** (console-only logging) | SotiAds | `SyncLog` + `ConfigSnapshot` persistence after every operation |
| **Undocumented internal RPCs** (fragile, breaks on UI update) | SotiAds `apis/admob.ts` | Official REST APIs only — stable, documented, versioned |
| **`pydash` for trivial array ops** | ironSource Python | Python builtins + list comprehensions |
| **Boolean encoding inconsistency** (0/1 vs 'active'/'inactive' vs true/false) | ironSource Python, LevelPlay API | Pydantic model validators normalize all API booleans to Python `bool` |

**Key insight**: The ironSource libs use `/instances/v3` while current Unity docs show `/instances/v1`. The libs' version is known to work. Test both but expect v3 to be correct.

## Tasks

### Task: Project Foundation

- **Type**: PoC
- **Validates**: Project structure and typed data models can correctly represent ad mediation configs (apps, tiers, groups, waterfalls, instances, placements, sync history) with clean abstractions that don't leak mediator-specific details
- **Unlocks**: LevelPlay Authentication (needs models and adapter interface)
- **Reference**: Study the ironSource lib's `MediationGroupPriority` (3-tier + bidders waterfall), `InstanceConfig` (per-network fields), `Placement` (capping/pacing), and enums (`AdUnits`, `Networks`, `TierType`, `Platform`) — these define the exact API vocabulary. Cross-reference with JS lib for the complete 29-network list. Study OM's `om_instance` and `om_placement_rule` for fields the ironSource lib doesn't model (hb_status, weight, sort_type, per-country eCPM).
- **Lessons applied**:
  - **Data-driven network config** — the ironSource lib has 42 classes for 33 networks (1549 lines of boilerplate). Instead: define per-network credential schemas as data (dict/registry), not one-class-per-network.
  - **Typed exceptions** — the ironSource lib uses generic `Exception` everywhere. Define: `AuthError`, `RateLimitError`, `ApiError`, `ValidationError` from the start.
  - **Boolean normalization** — the LevelPlay API uses `0`/`1` for some booleans (`adDelivery`, `coppa`, `capping.enabled`), `'active'`/`'inactive'` for status, and `true`/`false` elsewhere. Pydantic validators must normalize all of these to Python `bool`.
  - **Adapter capability flags** — the AdMob API has no mediation group CRUD (reporting only). Design `MediationAdapter` so read-only adapters are first-class: use a `capabilities` property or raise `NotImplementedError` with clear messages for unsupported operations.
  - **"All countries" sentinel** — OM uses `"00"` for all-countries fallback. TierTemplate must have an explicit default/catch-all tier.
  - **Waterfall validation** — the ironSource JS lib validates that `OPTIMIZED` tier type cannot coexist with bidders. Encode this as a pydantic model validator on `WaterfallConfig`.
- **Success Criteria**:
  - `pyproject.toml` installs cleanly with all dependencies (httpx, typer, fastmcp, pydantic, pyjwt, ruamel.yaml, python-dotenv, pytest, pytest-asyncio, ruff, mypy)
  - Package structure exists: `admedi/adapters/`, `admedi/engine/`, `admedi/cli/`, `admedi/mcp/`, `admedi/storage/`, `admedi/models/`
  - All 9 pydantic models defined with validation and serialization round-trips work: App, TierTemplate, Group, WaterfallConfig, Instance, Placement, SyncLog, ConfigSnapshot, Credential
  - Models handle API boolean encoding: `0`/`1` → `bool`, `'active'`/`'inactive'` → `bool`, camelCase API fields → snake_case Python fields (via `alias`)
  - Network credential schemas defined as a registry (dict mapping network name → required fields), not per-network classes
  - Typed exception hierarchy defined: `AdmediError` base → `AuthError`, `RateLimitError`, `ApiError(status_code, detail)`, `ValidationError`, `AdapterNotSupportedError`
  - MediationAdapter abstract interface defined with all 12 method signatures and a `capabilities` property: `authenticate()`, `list_apps()`, `get_groups()`, `create_group()`, `update_group()`, `delete_group()`, `get_instances()`, `create_instances()`, `update_instances()`, `delete_instance()`, `get_placements()`, `get_reporting()`
  - StorageAdapter abstract interface defined with all 5 method signatures: `save_config()`, `load_config()`, `save_sync_log()`, `list_sync_history()`, `save_snapshot()`
  - Unit tests pass for model serialization/deserialization (including edge cases: empty country lists, optional floor prices, nested WaterfallConfig, API boolean normalization, Optimized+Bidding incompatibility rejection)

### Task: LevelPlay Authentication

- **Type**: PoC
- **Validates**: We can authenticate with the LevelPlay REST API and discover Shelf Sort apps using real credentials
- **Unlocks**: LevelPlay Read & Write (needs working auth for all API calls)
- **Reference**: Study the ironSource lib's `base_api.py` (auth flow, credential caching, token invalidation on credential change) and `utils.py` (httpx setup, gzip, User-Agent). The lib's auth is proven to work — replicate the token lifecycle, but fix its shortcomings.
- **Lessons applied**:
  - **PyJWT, not manual base64** — the ironSource lib manually splits JWTs on `.` and base64-decodes segment [1]. Fragile. Use `PyJWT` with `options={"verify_signature": False}` to decode the `exp` claim properly.
  - **Persistent httpx.AsyncClient** — the ironSource lib creates and destroys a new client per request (no connection pooling). Create one `httpx.AsyncClient` per adapter instance, reuse across all calls.
  - **Timezone-aware datetimes** — the ironSource lib uses `datetime.utcnow()` (deprecated in Python 3.12+). Use `datetime.now(timezone.utc)` throughout.
  - **Lazy initialization** — the Google MCP Official repo loads credentials at import time (hard crash if env var missing). Initialize credentials on first `authenticate()` call, not at module import.
  - **Safety margin on token refresh** — the ironSource lib refreshes only when expired. Refresh when < 5 minutes remaining to avoid race conditions.
  - **Credential invalidation** — the ironSource lib correctly invalidates the cached token when credentials change. Preserve this pattern.
- **Success Criteria**:
  - OAuth 2.0 auth flow works: `GET /partners/publisher/auth` with `secretkey` (lowercase k) and `refreshToken` headers returns a valid Bearer token
  - JWT `exp` claim decoded via `PyJWT` — ironSource JWTs store `exp` in milliseconds (not standard seconds), handle this explicitly
  - Token is cached and auto-refreshes before the 60-minute expiry (refresh when `exp` claim < 5 minutes remaining)
  - Cached token invalidated automatically if credentials change (same pattern as ironSource lib's `set_credentials()`)
  - `list_apps()` returns all Shelf Sort apps with correct `appKey`, `appName`, `platform`, and `bundleId` — verified against LevelPlay dashboard
  - Credentials loaded from `.env` file via `python-dotenv` (`LEVELPLAY_SECRET_KEY`, `LEVELPLAY_REFRESH_TOKEN`)
  - Auth tokens are never logged — masked in all log output
  - `httpx.AsyncClient` configured as persistent session: 60s timeout, gzip support, connection pooling, custom User-Agent (`admedi/{version}`)
  - Auth errors raise `AuthError` (not generic `Exception`)

### Task: LevelPlay Read & Write Operations

- **Type**: Feature
- **Validates**: Full LevelPlay API coverage — we can read all mediation config data and modify configs safely
- **Unlocks**: ConfigEngine Pipeline (needs reads for diffing, writes for applying)
- **Reference**: Study the ironSource lib's `monetize_api.py` — all 18 endpoint methods show exact request body construction, response parsing, and which fields are required vs optional. Cross-reference with JS lib's `monetize_api.ts` for API quirks the Python lib misses. We add Groups API v4 and Reporting API v1 (not in either lib) plus rate limit handling and A/B test detection (not in either lib).
- **Lessons applied**:
  - **Per-app error isolation** — SotiAds wraps each app/format in try/catch so one failure doesn't abort the portfolio. Apply the same: concurrent multi-app reads must isolate failures per app.
  - **Exponential backoff** — neither ironSource lib has any retry logic. Build it in from the start: backoff on 429, refresh + retry on 401, retry with limit on 500.
  - **Rate limit tracking** — neither ironSource lib is aware of rate limits at all. Track request counts per endpoint window and proactively throttle before hitting limits.
  - **API quirks from JS lib** — delete placement sends `{appKey, adUnit, id}` in request **body** (not query params); placement update **cannot rename** (name field is immutable after creation); strip empty `appConfig` values before sending instance payloads.
  - **Batch failure handling** — the architecture doc notes "Instances API rejects entire batch if any single item fails." Validate each instance before sending, and on batch failure, retry items individually to identify the bad one.
- **Success Criteria**:
  - **Read endpoints all functional**:
    - `get_groups(app_key)` returns normalized Group models matching LevelPlay dashboard (via Groups API v4: `GET /levelPlay/groups/v4/{appKey}`)
    - `get_instances(app_key)` returns all ad network instances (bidding and non-bidding)
    - `get_placements(app_key)` returns all placements with capping/pacing config
    - `get_reporting(app_key, date_range, breakdowns)` returns eCPM, revenue, impressions data
  - **Write endpoints all functional**:
    - `create_group()`, `update_group()`, `delete_group()` work on a single test app without corrupting config
    - `create_instances()`, `update_instances()`, `delete_instance()` work on a single test app
    - Write operations verified by re-reading config after apply and comparing
  - **Known API quirks handled**: delete placement via body (not params), immutable placement names, empty appConfig stripping, instance batch rejection fallback to per-item retry
  - **Rate limit handling**: Exponential backoff on 429 responses, request counting to proactively throttle (Groups: 4K/30min, Instances: 8K/30min, Reporting: 8K/hr)
  - **A/B test detection**: Groups API response `abTest` field checked — apps with active A/B tests are flagged with clear warning, writes skipped for those apps
  - **Error handling**: 401 → token refresh + retry (`AuthError`); 429 → backoff (`RateLimitError`); 500 → retry with limit (`ApiError`); batch 400 → per-item retry and report
  - **Per-app isolation**: Failure on one app does not prevent operations on other apps — errors collected and reported per app
  - **API version resolution**: Groups v2 vs v4 and Instances v1 vs v3 tested — primary versions selected, fallbacks documented
  - Concurrent multi-app reads work (parallel `get_groups()` calls across app keys, respecting rate limits, isolated error handling per app)

### Task: ConfigEngine Pipeline

- **Type**: Feature
- **Validates**: Config-as-code works end-to-end — load a YAML tier template, diff against live remote config, and apply changes correctly
- **Unlocks**: CLI, MCP Server & Storage (needs engine methods to delegate to)
- **Reference**: Study SotiAds' `listChanges()` in `base.ts` for the generic diff pattern (but enhance with field-level diffs). Study SotiAds' `config.yml` for YAML schema design (but add country tiers, the feature SotiAds completely lacks). Study OM-Server's `Placement.java` and `Instance.java` for country fallback resolution patterns.
- **Lessons applied**:
  - **Generic diff pattern** — SotiAds' `listChanges(source, target, comparator)` → `{toAdd, toUpdate, toRemove}` is the right primitive. Enhance it: our Differ should produce field-level change details within `toUpdate` (what changed and from what to what), not just "these two items match."
  - **Dry-run is the default** — SotiAds has no dry-run mode; every sync is live. In Admedi, `diff` is the default operation. Applying changes requires explicit confirmation.
  - **Config snapshot before mutation** — SotiAds never snapshots remote state before writing. Admedi captures a `ConfigSnapshot` before every write so changes can be audited or manually reversed.
  - **Idempotency** — SotiAds always overwrites the entire mediation group regardless of changes. Admedi's Applier should produce zero API calls when re-running with no config changes (diff-based minimal writes).
  - **Country fallback resolution** — OM-Server consistently applies "try specific country → fall back to ALL." Admedi's TierTemplate must have an explicit default/catch-all tier, and the Differ must handle the "All Countries" group correctly.
  - **Template inheritance** — SotiAds duplicates floor lists across apps. Support YAML anchors/aliases or explicit template references so a tier template is defined once and shared across the portfolio.
- **Success Criteria**:
  - **YAML template**: Shelf Sort tier template created matching real dashboard config — tiers (Tier 1: US, Tier 2: AU/CA/DE/GB/JP/NZ/KR/TW, Tier 3: FR/NL, All Countries as catch-all), portfolio (6 apps x 3 platforms), ad formats (banner, interstitial, rewarded_video). Template supports YAML anchors for shared tier definitions.
  - **Loader**: Parses YAML via `ruamel.yaml` into validated TierTemplate via pydantic. Rejects invalid configs with clear error messages (bad country codes, duplicate countries across tiers, missing required fields, no catch-all tier)
  - **Differ**: Generic diff engine — `compute_diff(desired, current, matcher)` → structured DiffReport:
    - Correctly identifies missing countries (e.g., South Korea missing from Tier 2)
    - Correctly identifies countries that need removal (e.g., Malaysia in Tier 3)
    - Correctly identifies floor price mismatches
    - Correctly identifies groups that need creation (new tier not yet on remote)
    - DiffReport contains structured `additions[]`, `removals[]`, `modifications[]` per group per app
    - Each modification includes **field-level detail**: what changed, old value, new value, human-readable description
    - Empty DiffReport (no changes) is a valid result — means remote matches template
  - **Applier**: Executes DiffReport by calling adapter write methods:
    - Atomic per-app: failure on one app doesn't affect others (try/except per app)
    - **Dry-run is the default**: generates DiffReport preview without making any API calls
    - **Snapshot before write**: captures `ConfigSnapshot` of remote state before any mutations
    - **Idempotent**: re-running with no config changes produces zero API calls (empty DiffReport → no writes)
    - Applier verifies writes by re-reading config after apply
  - **End-to-end**: Load Shelf Sort template → diff against one real app → verify DiffReport accuracy manually → apply to one test app → re-audit shows zero drift
  - Unit tests for Differ with mock data (known template vs known remote → expected DiffReport, including empty-diff case)

### Task: CLI, MCP Server & Storage

- **Type**: Feature
- **Validates**: All user-facing interfaces work correctly and persist state — users can interact with Admedi via terminal, AI agents, or programmatic import
- **Unlocks**: Portfolio Dogfood (needs interfaces for daily use)
- **Reference**: Study Google MCP GMS repo for MCP patterns: `coordinator.py` (singleton), `tools/api.py` (`ToolError`, `output_schema`), `tools/docs.py` (MCP Resources), `server.py`/`stdio.py` (dual transport). Study SotiAds' CLI for sync/list command design (but add dry-run, diff-only, and richer output).
- **Lessons applied**:
  - **Coordinator singleton** — Google MCP GMS pattern: `FastMCP` instance in dedicated `coordinator.py`, imported by tool modules via decorator side effects. Cleanly separates server instantiation from tool registration.
  - **Standalone `fastmcp`** — use the standalone `fastmcp>=3.0.2` package (not `mcp` package's bundled version). Provides `ToolError`, `output_schema`, `mask_error_details`, auth providers, and multiple transport support.
  - **`ToolError` for user-facing errors** — Google MCP GMS wraps API exceptions in `ToolError` with human-readable messages. Combined with `mask_error_details=True` to hide sensitive details from MCP clients.
  - **`output_schema` on structured tools** — Google MCP GMS uses explicit JSON Schema for return types. Apply to `get_groups`, `audit`, `status` tools so LLM clients know the response structure.
  - **MCP Resources** — Google MCP GMS exposes docs as both tools and resources. Expose tier template definitions and portfolio status as MCP resources for passive access.
  - **Two-phase writes** — neither Google MCP repo has write operations (both are read-only). Admedi must design its own pattern: `sync_tiers` returns a DiffReport preview first; a second call with `confirm=true` executes it.
  - **Lazy credentials** — Google MCP Official loads credentials at import time (crashes hard if env var missing). Initialize credentials on first tool call, not at server startup.
- **Success Criteria**:
  - **CLI (typer)** — all 5 commands functional:
    - `admedi sync-tiers` — loads template, diffs against remote, shows preview table, applies on confirm
    - `admedi audit` — read-only drift detection with formatted table output
    - `admedi revenue` — pulls reporting data, shows per-tier eCPM averages
    - `admedi manage-instances` — bulk instance operations (add, remove, enable, disable)
    - `admedi status` — portfolio overview (apps, platforms, last sync time)
    - Flags work: `--dry-run`, `--app` (single app target), `--format json`, `--config` (template path)
    - Exit codes: 0 = success, 1 = drift detected, 2 = error
    - Rich terminal output: colored diffs, formatted tables, progress indicators for multi-app operations
  - **MCP Server (FastMCP)** — all 6 tools functional:
    - `get_groups(app_key)`, `sync_tiers(template_path, app_keys?)`, `audit(template_path, app_keys?)`, `revenue_check(app_keys, days?)`, `manage_instances(action, network, app_keys?)`, `status()`
    - Coordinator singleton pattern: `admedi/mcp/coordinator.py` → `FastMCP("Admedi Server", mask_error_details=True)`
    - `output_schema` on tools returning structured data (`get_groups`, `audit`, `status`)
    - API exceptions wrapped in `ToolError` with user-friendly messages
    - MCP Resources: expose tier template schema and portfolio summary as `@mcp.resource()`
    - Tools callable from Claude Code with correct JSON schema parameters
    - Generic tool names (no mediator prefix)
    - Credentials loaded lazily on first tool call — not at server import, not transmitted over MCP protocol
    - Write operations are two-phase: first call returns DiffReport preview, second call with `confirm=true` executes
  - **Local File Storage Adapter** — all 5 methods functional:
    - `save_config()`, `load_config()`, `save_sync_log()`, `list_sync_history()`, `save_snapshot()`
    - `.admedi/` directory structure created: `configs/`, `logs/`, `snapshots/`
    - Sync logs are append-only JSON lines (one line per operation)
    - Snapshots stored as `{app_key}_{timestamp}.json`
  - Sync logs persisted in `.admedi/logs/` after every sync operation
  - Config snapshots captured in `.admedi/snapshots/` after every sync
  - Fully mocked tests for MCP tools — mock the LevelPlayAdapter, no live credentials required (Google MCP GMS test pattern)

### Task: Portfolio Dogfood

- **Type**: Feature
- **Validates**: Admedi works on real production data for daily ad ops — the tool actually saves time and produces correct results on the Shelf Sort portfolio
- **Unlocks**: Milestone complete — ready for open-source milestone
- **Lessons applied**:
  - **Tagged per-app logging** — SotiAds uses `consola.withTag(app.appId)` for per-app log context. Apply the same: Python `logging` with structured context (app_key, platform, operation) so multi-app output is easy to follow.
  - **Continuation from failure** — SotiAds processes apps sequentially and if one fails, continues to the next. Admedi should report partial failures (which apps succeeded, which failed and why) and support resuming from the last failure point.
- **Success Criteria**:
  - Production Shelf Sort YAML tier template created from current LevelPlay dashboard config (bottom-up: export real config, build YAML from it)
  - Initial `admedi audit` run identifies all existing config drift across 18 surfaces
  - First `admedi sync-tiers` applies template across all 18 surfaces — re-audit shows zero drift
  - Full portfolio sync completes in < 2 minutes
  - 2+ weeks of daily use with zero manual dashboard configuration for tier changes
  - At least one real tier adjustment (country promotion or demotion) applied via `admedi sync-tiers`
  - MCP tools used successfully via Claude Code for at least 3 different operations (e.g., audit, status, revenue check)
  - All bugs discovered during dogfood documented and fixed
  - OAuth token auto-refresh works across multi-hour sessions with zero auth failures
  - Zero hard 429 rate limit errors during normal operations
  - Per-app log output is clear and traceable (structured logging with app_key context)
  - Partial portfolio failures are reported cleanly — which apps succeeded, which failed, and why

## Execution Order

1. Task: Project Foundation (no dependencies)
2. Task: LevelPlay Authentication (requires Project Foundation)
3. Task: LevelPlay Read & Write Operations (requires LevelPlay Authentication)
4. Task: ConfigEngine Pipeline (requires LevelPlay Read & Write Operations)
5. Task: CLI, MCP Server & Storage (requires ConfigEngine Pipeline)
6. Task: Portfolio Dogfood (requires CLI, MCP Server & Storage)

## Integration Points

Tasks integrate through the layered architecture:

- **Project Foundation → LevelPlay Authentication**: Auth implements the `authenticate()` and `list_apps()` methods of the MediationAdapter interface defined in Foundation. Uses pydantic Credential and App models.
- **LevelPlay Authentication → Read & Write**: All subsequent adapter methods use the token management established in Authentication. Same `httpx.AsyncClient` instance with auth headers injected.
- **Read & Write → ConfigEngine**: The Differ calls `get_groups()` to pull remote state for comparison. The Applier calls `create_group()`, `update_group()`, `delete_group()` to execute diffs. All data flows through the pydantic models (Group, WaterfallConfig, DiffReport).
- **ConfigEngine → Interfaces**: CLI commands and MCP tools are thin wrappers around ConfigEngine methods (`sync_tiers()`, `audit()`, `revenue()`, etc.). The Local Storage Adapter is called by the engine after each sync to persist logs and snapshots.
- **Interfaces → Dogfood**: Dogfood uses the CLI and MCP server as built. No new integration — this is pure validation on real data.

## Risk Assessment

| Task | Risk Level | Mitigation |
|------|------------|------------|
| Task: Project Foundation | L | Standard scaffolding — well-understood patterns. Risk is only in data model accuracy, mitigated by basing models on real LevelPlay API responses from both ironSource libs. Network schema registry is novel but lower risk than 42-class approach. |
| Task: LevelPlay Authentication | H | Highest risk task — if auth doesn't work, everything stops. Unity's API docs may have gaps. Mitigate by replicating the proven ironSource lib auth flow (JWT `exp` claim, `secretkey` header), testing against real API immediately. The ironSource lib's auth is confirmed working — we replicate the logic, fix the implementation. |
| Task: LevelPlay Read & Write Operations | M | API behavior may differ from docs (known Unity issue). Rate limits could be tighter than documented. API version discrepancies (v1 vs v3 Instances) need live testing. Mitigate by testing every endpoint individually, logging raw responses, comparing against dashboard. Reference both ironSource libs for request/response formats. Batch instance API rejection is a known gotcha — handle with per-item fallback. |
| Task: ConfigEngine Pipeline | M | YAML template format may not capture all real-world config complexity. Differ may miss edge cases. Country fallback (specific → ALL) must be correct. Mitigate by building template bottom-up from real Shelf Sort config, testing Differ with diverse mock data including empty-diff case (idempotency). |
| Task: CLI, MCP Server & Storage | L | Standard interface work — typer and FastMCP are well-documented with strong reference implementations (Google MCP repos). Two-phase write confirmation for MCP is novel but straightforward. Storage adapter is simple flat files. |
| Task: Portfolio Dogfood | M | Real production data may expose edge cases not caught in testing. A/B tests on Shelf Sort apps could block API calls. Mitigate by running dry-run first (it's the default), capturing snapshots before every apply, per-app error isolation with partial failure reporting. |

## Feedback Loops

### If a Task Fails

**A failed task is valuable information, not wasted effort.**

When a task doesn't meet success criteria:

1. **Document what we learned** — What specifically failed? Why?
2. **Assess impact** — Does this invalidate the milestone approach? Or just this task?
3. **Decide next action**:
   - **Retry with different approach** — Update task design and re-attempt
   - **Pivot the milestone** — Revisit milestone spec with new constraints
   - **Revisit architecture** — If fundamental assumption was wrong
   - **Kill the milestone** — If the capability isn't achievable/valuable

### Checkpoint Questions

After each task, ask:
- Did we learn something that changes our assumptions?
- Should we update subsequent task designs based on this learning?
- Is the milestone still viable and valuable?

### Critical Checkpoints

**After LevelPlay Authentication**: If auth fails or the API is inaccessible, the entire milestone is blocked. This is the go/no-go gate. If the existing ironSource lib's auth pattern doesn't work against the current API, investigate alternative auth methods before proceeding.

**After ConfigEngine Pipeline**: If the Differ can't produce accurate diffs for real Shelf Sort configs, the config-as-code premise is invalidated. This is the second go/no-go gate. If the YAML template can't represent real config complexity, revisit the template format before building interfaces.

**After Portfolio Dogfood (first week)**: If daily usage reveals fundamental issues (corrupted configs, missed changes, unreliable sync), extend dogfood and fix before declaring milestone complete. Do not rush to the next milestone with unresolved reliability issues.
