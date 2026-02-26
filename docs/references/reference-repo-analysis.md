# Reference Repository Analysis

**Date**: 2026-02-25
**Purpose**: Detailed analysis of 8 reference repositories cloned to `references/`, documenting their usefulness, patterns worth adopting, and relevance to each Admedi task.

## Table of Contents

1. [Executive Summary](#executive-summary)
2. [Repository Inventory](#repository-inventory)
3. [ironSource Python Library](#1-ironsource-python-library)
4. [ironSource JS Library](#2-ironsource-js-library)
5. [SotiAds](#3-sotiads)
6. [Google Ads MCP — GMS](#4-google-ads-mcp--gms)
7. [Google Ads MCP — Official](#5-google-ads-mcp--official)
8. [OpenMediation](#6-openmediation)
9. [OpenMediation Server](#7-openmediation-server)
10. [AdMob API Samples](#8-admob-api-samples)
11. [Cross-Repository Patterns](#cross-repository-patterns)
12. [Entity Mapping Across Repos](#entity-mapping-across-repos)
13. [Per-Task Reference Guide](#per-task-reference-guide)
14. [Key Findings](#key-findings)

---

## Executive Summary

We analyzed 8 open-source repositories across 5 categories to inform Admedi's implementation. The analysis reveals that **Admedi will be the first open-source tool to wrap LevelPlay CRUD APIs** (Groups v4, Instances, Placements) in any language. No wrapper exists for MAX either. This is both a market gap and an implementation challenge — there are no community reference implementations to validate against.

**Top findings**:
- The ironSource Python + JS libraries are the definitive source for API field names, auth flow, and endpoint behavior
- SotiAds validates the YAML-config-to-API-sync concept but uses fragile undocumented AdMob RPCs
- Google's MCP server repos provide excellent FastMCP patterns (coordinator singleton, ToolError, output_schema)
- OpenMediation has the most complete multi-network data model (27+ tables) for reference
- AdMob's public API is reporting-only — no mediation group CRUD exists, limiting any future AdMob adapter to read/audit

---

## Repository Inventory

| # | Repo | Local Path | Language | License | Stars | Last Active | Relevance |
|---|------|-----------|----------|---------|-------|-------------|-----------|
| 1 | ironSource/mobile-api-lib-python | `references/ironsource-python-lib/` | Python | Apache-2.0 | 1 | 2023-03 | **Critical** |
| 2 | ironSource/mobile-api-lib-js | `references/ironsource-js-lib/` | TypeScript | Apache-2.0 | 0 | 2023-03 | High |
| 3 | shtse8/SotiAds | `references/sotiads/` | TypeScript | Unlicensed | 3 | 2025-04 | High |
| 4 | google-marketing-solutions/google_ads_mcp | `references/google-ads-mcp-gms/` | Python | Apache-2.0 | 109 | 2026-02 | High |
| 5 | googleads/google-ads-mcp | `references/google-ads-mcp-official/` | Python | Apache-2.0 | 243 | 2025-10 | Medium |
| 6 | OpenMediationProject/OpenMediation | `references/openmediation/` | Meta-repo | LGPL-3.0 | 106 | 2021-11 | Medium |
| 7 | OpenMediationProject/OM-Server | `references/openmediation-server/` | Java | LGPL-3.0 | 15 | 2022-10 | Medium |
| 8 | googleads/googleads-admob-api-samples | `references/admob-api-samples/` | Java/Python | Apache-2.0 | 47 | 2024-01 | Low (future) |

---

## 1. ironSource Python Library

**Path**: `references/ironsource-python-lib/`
**Relevance**: Critical — primary reference for auth flow, API field names, and endpoint behavior.

### Structure

```
ironsource_api/
├── __init__.py              # Version: 1.1.0
├── base_api.py              # Auth: JWT caching, credential storage
├── utils.py                 # HTTP client: httpx, gzip, pagination
├── ironsource_api.py        # Facade: MonetizeAPI + PromoteAPI
└── monetize_api/
    ├── __init__.py           # Enums: Networks(33), AdUnits, Metrics(26), Breakdowns(15)
    ├── monetize_api.py       # 18 endpoint methods, URL constants
    ├── mediation_group_priority.py  # 3-tier + bidders waterfall model
    ├── instance_config.py    # 42 network-specific config subclasses (1549 lines)
    └── placement_config.py   # Placement, Capping, Pacing models
```

### Auth Flow (`base_api.py`)

The proven auth pattern:

1. Store `secretKey`, `refreshToken`, cached JWT, and expiration timestamp
2. Before each API call, check if cached JWT is still valid (exp > now)
3. If expired, `GET /partners/publisher/auth` with headers `secretkey` (lowercase k) and `refreshToken`
4. Parse JWT payload (base64 decode segment [1]), extract `exp` claim
5. Cache both the token and expiration for future calls

**JWT gotcha**: ironSource JWTs store `exp` in **milliseconds** (not standard seconds). The code divides by 1000 before comparison.

**What Admedi should do differently**:
- Use `PyJWT` library instead of manual base64 splitting
- Add a safety margin (refresh 5 minutes before actual expiry)
- Use pydantic for credential models instead of private class variables
- Use timezone-aware datetimes (`utcnow()` is deprecated in Python 3.12+)

### HTTP Client (`utils.py`)

- Auth URL: `https://platform.ironsrc.com/partners/publisher/auth`
- Creates a new `httpx.AsyncClient` per request (no connection pooling — a bug)
- 60-second timeout, gzip support, custom User-Agent
- Returns `ResponseInterface(msg, error_code)` where `-1` = success
- Pagination: follows `paging.next` for JSON, `Link` header for CSV

**What Admedi should do differently**:
- Use a **persistent** `httpx.AsyncClient` with connection pooling
- Implement retry with exponential backoff
- Raise typed exceptions instead of generic `ResponseInterface`
- Use async generators for pagination instead of OS pipes + threads

### URL Constants (Confirmed Working Endpoints)

| Constant | URL | Version |
|----------|-----|---------|
| Auth | `https://platform.ironsrc.com/partners/publisher/auth` | — |
| Apps | `/partners/publisher/applications/v6` | v6 |
| Mediation Groups | `/partners/publisher/mediation/management/v2` | v2 (legacy) |
| Instances | `/partners/publisher/instances/v3` | v3 |
| Placements | `/partners/publisher/placements/v1` | v1 |
| Reporting | `/partners/publisher/mediation/applications/v6/stats` | v6 |

**Note**: Admedi targets the newer LevelPlay Groups API v4 (`/levelPlay/groups/v4/{appKey}`), not the v2 management endpoint. The v2 endpoints are historical context only.

### Enum Values (Exact API Vocabulary)

**Networks** (33): ironSource, ironSourceBidding, AppLovin, AdColony, adColonyBidding, AdMob, AdManager, Amazon, Chartboost, crossPromotionBidding, CSJ, DirectDeals, Facebook, facebookBidding, Fyber, HyprMX, InMobi, inMobiBidding, Liftoff Bidding, Maio, mediaBrix, myTargetBidding, Pangle, pangleBidding, smaatoBidding, Snap, SuperAwesomeBidding, TapJoy, TapJoyBidding, Tencent, UnityAds, Vungle, vungleBidding, yahooBidding

**AdUnits**: `rewardedVideo`, `interstitial`, `banner`, `OfferWall` (note capital W)

**AdUnitStatus**: `Live`, `Off`, `Test`

**Platform**: `iOS`, `Android` (exact casing)

**TierType**: `manual`, `sortByCpm`, `optimized`, `bidding`

### Waterfall Data Model (`mediation_group_priority.py`)

The mediation group priority structure: 3 tiers + separate bidders group.

```
adSourcePriority: {
    tier1: { tierType: 'manual'|'sortByCpm'|'optimized', instances: [...] },
    tier2: { tierType: ..., instances: [...] },
    tier3: { tierType: ..., instances: [...] },
    bidding: { tierType: 'bidding', instances: [...] }
}
```

Each instance in a tier: `{ providerName: str, instanceId: int, rate?: int, capping?: int }`

**Bugs found in reference**:
- `_validate_tier` uses bare `position` and `list` (builtin) instead of string keys in error dict
- `update_mediation_group` unconditionally accesses `ad_source_priority.get_object()` even when None

### Instance Config (`instance_config.py`)

42 network-specific subclasses (1549 lines of mostly boilerplate). Each defines:
- `get_app_data_obj()` → network-level config (`appConfig` key)
- `get_object()` → instance-level config per ad unit

**What Admedi should do differently**: Use a single generic `InstanceConfig` pydantic model with a `network_fields: dict` property, or a network schema registry defined as data (YAML/JSON) rather than 42 Python classes.

### API Field Names (camelCase, exact)

**App**: `appKey`, `appName`, `adUnits`, `appStatus`, `bundleId`, `creationDate`, `icon`, `networkReportingApi`, `platform`, `coppa`, `taxonomy`, `bundleRefId`, `ccpa`

**Instance**: `instanceName`, `instanceId`, `status` ('active'/'inactive'), `rate`, `pricing` (array of `{eCPM, Countries}`)

**Mediation Group**: `appKey`, `adUnit`, `groupName`, `groupId`, `groupCountries`, `groupPosition`, `groupSegments`, `adSourcePriority`

**Placement**: `appKey`, `adUnit`, `name`, `id`, `adDelivery` (0/1), `itemName`, `rewardAmount`, `capping`, `pacing`, `abVersion`

**Boolean encoding**: API uses `0`/`1` integers for some booleans (`adDelivery`, `coppa`, `ccpa`, `capping.enabled`), `'active'`/`'inactive'` strings for status.

### Test Patterns

- Mock `BaseAPI.get_bearer_auth` to return `'TOKEN'`
- Mock `execute_request` to return controlled `ResponseInterface`
- Assert correct URL, method, and options were passed
- Integration tests use real API calls with `pytest-ordering` for sequencing

---

## 2. ironSource JS Library

**Path**: `references/ironsource-js-lib/`
**Relevance**: High — TypeScript type definitions cross-reference pydantic models; more up-to-date network list.

### Key Differences from Python Lib

The JS lib mirrors the Python lib's architecture but reveals additional details:

**Networks** (29 in JS vs 33 in Python — different counting due to bidding variants):
Networks added after Python lib was abandoned: `Liftoff`, `MyTarget`, `Smaato`, `Snap`, `SuperAwesome`, `Yahoo`, `Tencent`, `CSJ`. Admedi should include all of these.

**26 InstanceConfig subclasses** (vs 42 in Python) — same networks, fewer subclasses due to TypeScript's more compact inheritance.

**API version confirmation**: Both libs agree on `/instances/v3`, confirming this is the correct version (not v1 as shown in some Unity docs).

### Notable API Behaviors Revealed

1. **ARM/UAR two-step fetch**: Getting impression-level revenue requires calling the API to get a signed S3 URL, then fetching + decompressing. Response: `{urls: ["https://..."], expiration: "..."}`

2. **Delete placement uses body, not query params**: `DELETE /placements/v1` sends `{appKey, adUnit, id}` in the request body

3. **Instance appConfig empty check**: When all appConfig values are empty strings, strip the appConfig key before sending

4. **Placement update ignores name**: API does not allow name changes on update

5. **Optimized + Bidding incompatibility**: `OPTIMIZED` tier type cannot coexist with bidders — business rule validated in code

6. **Bid chunk size limit**: 9,998 items per batch (undocumented API limit)

### Metrics Comparison

JS lib has 24 metrics (targeting v6/stats endpoint). The newer LevelPlay Reporting v1 endpoint (from our API reference doc) has a smaller set. The v6 endpoint includes granular metrics not in v1: `completions`, `clicks`, `clickThroughRate`, `appFillRate`, `appRequests`, `adSourceChecks/Responses/AvailabilityRate`.

### Platform Casing Gotcha

- Monetize API: `iOS`, `Android` (mixed case)
- Promote API: `ios`, `android` (lowercase)

Admedi should normalize to a single casing in its models.

---

## 3. SotiAds

**Path**: `references/sotiads/`
**Relevance**: High — closest existing tool to Admedi. Validates the YAML-config-to-API-sync concept.

### Architecture

```
sotiads/
├── config.yml            # YAML source of truth
└── src/
    ├── index.ts          # CLI: commander, sync + list commands
    ├── base.ts           # Enums, interfaces, listChanges() diff utility
    ├── read.ts           # YAML config loader (valibot schema)
    └── apis/
        ├── admob.ts      # AdMob API (undocumented internal RPCs)
        ├── firebase.ts   # Firebase Remote Config updater
        └── google.ts     # Playwright browser-based auth
```

### YAML Config Schema

```yaml
default:
  ecpmFloors: [15.0, 10.0, 5.0, 2.0]    # Global fallback floors

apps:
  ca-app-pub-XXX~YYY:                      # Key = AdMob app ID
    placements:
      default:                              # Logical placement name
        interstitial:
          ecpmFloors: [20.0, 15.0, 10.0]   # Per-format override
        rewarded:
          ecpmFloors: []                    # Empty = use default
    adSources:
      meta:
        appId: "123456789"
        placements:
          default:
            interstitial:
              placementId: "IMG_16_9_APP_INSTALL#..."
```

**Key schema observations**:
- Two-level nesting: `placement-id` → `ad-format` → config
- eCPM floors are per-format, per-placement (not just per-app)
- Fallback hierarchy: format-level → `default.ecpmFloors`
- Ad sources mirror the placement structure with network-specific credentials
- **No country/geo tiers** — operates purely on eCPM price floors (major gap vs Admedi)

### The `listChanges()` Diff Utility

The core diff function — a generic three-way diff producing `{ toAdd, toUpdate, toRemove }`:

```typescript
function listChanges<S, T>(
    source: S[],             // desired state (from YAML)
    target: T[],             // current state (from API)
    comparator: (a: S, b: T) => boolean  // matching function
): { toAdd: S[], toUpdate: [S, T][], toRemove: T[] }
```

**Algorithm**: Build count map of source items → iterate target finding matches → remainder = toAdd/toRemove.

**What Admedi should adopt and enhance**:
- The generic `(source, target, comparator) → {toAdd, toUpdate, toRemove}` pattern maps directly to Admedi's Differ concept
- Enhance with: field-level diffs in `toUpdate`, human-readable summary for dry-run, change metadata

### Sync Flow

For each app → each placement → each ad format:
1. `syncAdUnits()` — create/update/delete ad units to match eCPM floors
2. `syncMediationGroup()` — create/update mediation group with all units + sources
3. `updateAdUnits()` on Firebase — push ad unit IDs into Remote Config

### Critical Weakness: Undocumented Internal RPCs

SotiAds does NOT use the official AdMob API. It reverse-engineers AdMob web console internal endpoints:

| Endpoint | Purpose |
|----------|---------|
| `PublisherService/Get` | Get publisher info |
| `AdUnitService/List/Create/Update/BulkRemove` | Ad unit CRUD |
| `MediationGroupService/List/V2Create/V2Update` | Group CRUD |
| `MediationAllocationService/Update` | Ad source → unit mappings |

Auth is done via **Playwright browser automation** — launching real Chromium, waiting for manual login, scraping session cookies and XSRF tokens.

### What Admedi Should Do Differently

| Aspect | SotiAds | Admedi |
|--------|---------|----------|
| API access | Undocumented internal RPCs (fragile) | Official REST APIs (stable) |
| Dry-run mode | None — every sync is live | Core feature (`diff` command) |
| Geo targeting | None | First-class tier system |
| Multi-app processing | Sequential | Async parallel with rate limiting |
| Adapter pattern | Hardcoded to AdMob | Formal `MediationAdapter` interface |
| Schema validation | Defined but unused (cast) | Pydantic validates on construction |
| Audit trail | Console output only | `SyncLog` + `ConfigSnapshot` |
| Error recovery | No retry, no rate limits | Exponential backoff, continuation |
| Config snapshots | None (no rollback) | Snapshot before mutation |
| Template inheritance | Duplicated floor lists | YAML anchors or template references |

### Patterns Worth Adopting

1. **`listChanges()` diff primitive** — generic comparator-based diff
2. **YAML as single source of truth** — same core concept as Admedi
3. **Structured naming conventions** — `{placementId}/{adFormat}/{ecpmFloor}` for matching
4. **Per-app error isolation** — one failed format doesn't abort the app
5. **Default/override hierarchy** — extend to `default → per-app → per-placement → per-format → per-tier`
6. **Tagged logging** — `consola.withTag(app.appId)` → Python `logging` with per-app context

---

## 4. Google Ads MCP — GMS

**Path**: `references/google-ads-mcp-gms/`
**Relevance**: High — most mature MCP server reference. Actively maintained (last push 2026-02-25).

### Structure

```
ads_mcp/
├── coordinator.py       # Singleton FastMCP with mask_error_details=True
├── server.py            # HTTP transport entry point (OAuth)
├── stdio.py             # stdio transport entry point (no auth)
├── utils.py             # MODULE_DIR, ROOT_DIR constants
├── tools/
│   ├── api.py           # list_accessible_accounts + execute_gaql (with output_schema)
│   └── docs.py          # 3 doc tools + 3 MCP resources
├── context/             # Bundled documentation (GAQL.md, views.yaml, etc.)
└── scripts/
    └── generate_views.py  # Auto-generate view YAML docs
```

### Framework: standalone `fastmcp>=3.0.2`

Uses the standalone FastMCP package (not the `mcp` package's bundled version):

```python
from fastmcp import FastMCP
mcp_server = FastMCP(name="Google Ads API", mask_error_details=True)
```

Key features available: `ToolError`, `output_schema`, `get_access_token()`, auth providers, multiple transport support, MCP Resources.

### Tool Definition Patterns

**With explicit output schema**:
```python
@mcp.tool(
    output_schema={
        "type": "object",
        "properties": {
            "data": {"type": "array", "items": {"type": "object"}},
        },
        "required": ["data"],
    }
)
def execute_gaql(query: str, customer_id: str, login_customer_id: str | None = None) -> list[dict]:
```

**Simple decorator**:
```python
@mcp.tool()
def list_accessible_accounts() -> list[str]:
    """Lists Google Ads customers directly accessible."""
```

**MCP Resources** (passive data endpoints alongside tools):
```python
@mcp.resource("resource://Google_Ads_Query_Language")
def get_gaql_doc_resource() -> str:
```

### Error Handling

```python
from fastmcp.exceptions import ToolError

try:
    result = client.search(...)
except GoogleAdsException as e:
    raise ToolError("\n".join(str(i) for i in e.failure.errors)) from e
```

Combined with `mask_error_details=True` on the coordinator — raw error details hidden from MCP client in production.

### Credential Management

Multi-layered strategy:
1. **MCP-level OAuth** via `GoogleProvider` (full OAuth flow)
2. **Token verification** via `GoogleTokenVerifier`
3. **YAML file credentials** (`google-ads.yaml` with `developer_token`, `client_id`, etc.)
4. `python-dotenv` for `.env` loading
5. Lazy initialization (not at import time)
6. Request-scoped access tokens via `fastmcp.server.dependencies.get_access_token()`

### Testing Patterns

- `pytest` + `pytest-asyncio` + `pytest-mock`
- Fully mocked — no live credentials needed
- `reset_ads_client` fixture to reset cached state between tests
- Parametrized tests for input validation
- `mock.mock_open` for file-based tests

### Recommended Patterns for Admedi

| Pattern | How to Apply |
|---------|-------------|
| **Coordinator singleton** | `admedi/mcp/coordinator.py` with `FastMCP("Admedi Server", mask_error_details=True)` |
| **`ToolError` for errors** | Wrap `LevelPlayAPIError` in `ToolError` with user-friendly messages |
| **`output_schema`** | Use for structured returns from `get_groups`, `audit` tools |
| **MCP Resources** | Expose tier templates and API reference as resources |
| **Lazy credential init** | Don't create API client at import time |
| **Dual transport** | stdio for CLI, streamable-http for remote |
| **Fully mocked tests** | Mock `LevelPlayAdapter` in all tool tests |

---

## 5. Google Ads MCP — Official

**Path**: `references/google-ads-mcp-official/`
**Relevance**: Medium — simpler implementation, useful as a minimal MCP reference.

### Key Differences from GMS

| Aspect | Official | GMS |
|--------|----------|-----|
| MCP framework | `mcp[cli]>=1.2.0` (bundled FastMCP) | `fastmcp>=3.0.2` (standalone) |
| Tools | 2 (list_customers, search) | 4 + 3 resources |
| Transport | stdio only | stdio + streamable-http |
| Error handling | None (raw propagation) | `ToolError` + masked errors |
| Output schema | None | Explicit JSON Schema |
| Credentials | ADC at import time | YAML + OAuth + lazy init |
| Testing | Smoke tests, needs live creds | Fully mocked, CI-friendly |

### Notable Pattern: Dynamic Description Embedding

```python
mcp.add_tool(
    search,
    title="Fetches data from Google Ads API",
    description=_search_tool_description(),  # Embeds large reference file
)
```

The `_search_tool_description()` function embeds a large JSON reference file directly into the tool description, giving the LLM all context needed to construct valid queries without additional tool calls. Consider this for Admedi tools that need embedded reference data.

### Side-Effect Registration Pattern

```python
# server.py — imports register tools via module-level decorators
from ads_mcp.tools import search, core  # noqa: F401
```

Both repos agree on this pattern: tool modules are imported purely for their `@mcp.tool()` decorator side effects.

---

## 6. OpenMediation

**Path**: `references/openmediation/`
**Relevance**: Medium — best multi-network data model reference. Full mediation platform architecture.

### Sub-Project Architecture

OpenMediation consists of 7 interdependent sub-projects:

| Sub-Project | Role |
|---|---|
| **OM-Server** | SDK-facing mediation server (waterfall, bidding, events) |
| **OM-DTask** | Config center + Athena data processing |
| **OM-ADC** | Revenue data aggregation from ad network APIs |
| **OM-Dashboard-Server** | Dashboard REST backend (CRUD for all entities) |
| **OM-Dashboard-UI** | React frontend |
| **OM-Android-SDK / OM-iOS-SDK** | Client SDKs |

**Data flow**: Dashboard writes to MySQL → OM-DTask serializes to protobuf `.gz` → OM-Server rsyncs every 60s → in-memory cache.

### Database Schema (55+ Tables)

The meta-repo contains `om-init.sql.gz` with the complete MySQL schema. Key entity tables:

| Table | Maps to Admedi | Key Fields |
|-------|-----------------|------------|
| `om_publisher` | (none until SaaS) | id, name, status, domain, token |
| `om_publisher_app` | **App** | id, plat, app_key, bundle_id, available_countries, coppa |
| `om_placement` | **Placement** | id, ad_type(0-5), floor_price, hb_status, frequency_cap, device targeting |
| `om_placement_country` | (per-country Placement) | floor_price, max_price, time-of-day scheduling |
| `om_adnetwork` | (implicit in adapter) | id, name, type(bitmask), sdk_version |
| `om_adnetwork_app` | **Credential** (partial) | pub_app_id, adn_id, client_id, client_secret, refresh_token |
| `om_instance` | **Instance** | id, adn_id, placement_key, weight, hb_status, frequency_cap |
| `om_instance_country` | (per-country Instance) | ecpm (manual), time-of-day scheduling |
| `om_placement_rule` | **Group** | segment targeting, sort_type, auto_opt, priority |
| `om_placement_rule_segment` | **TierTemplate** | countries, frequency, brand/model targeting |
| `om_placement_rule_instance` | **WaterfallConfig** | instance priority/weight per rule |
| `om_placement_abt` | (A/B test tracking) | a_rule_id, b_rule_id, a_per, b_per, dates |
| `om_country` | (inline in TierTemplate) | a2, a3, tier, continent, dcenter |

### Ad Type Enum

OM uses numeric IDs: 0=Banner, 1=Native, 2=RewardedVideo, 3=Interstitial, 4=Splash, 5=CrossPromotion.

### Waterfall Priority Resolution

Multi-layer system:
1. **Country matching**: Rules indexed by `(placementId, country)`, fall back to "00" (ALL)
2. **Rule matching**: Sorted by priority (ascending), first matching segment wins
3. **Within a rule**: Groups ordered by `groupLevel` (tier level)
   - Auto mode (`autoSwitch=1`): sorted by eCPM
   - Manual mode: sorted by configured priority
   - Weight mode (`sortType=0`): weighted random
4. **Bidding instances**: extracted separately, compared by eCPM, inserted at appropriate position

### eCPM Algorithms

Two algorithms selectable per-rule:
- **Algorithm 1**: Cascading fallback — instance-country eCPM at 3h/6h/12h/24h windows → network-country → network-global
- **Algorithm 2**: Exponential smoothing — 7 days of daily eCPM, coefficient of variation selects smoothing level (0.9/0.6/0.3)

### A/B Testing

Two levels:
- **Rule-level**: `abs(device.hashCode()) % 100 < aPer` for deterministic split. Each variant has its own groups with instance priorities.
- **Placement-level**: Compares two complete rules with traffic percentages and date ranges.

---

## 7. OpenMediation Server

**Path**: `references/openmediation-server/`
**Relevance**: Medium — runtime mediation server with rich data model implementations.

### Key Java Classes

| Class | What It Models | Relevant Fields |
|-------|---------------|----------------|
| `PublisherApp.java` | App with block rules | app_key, available_countries, UAR data |
| `Placement.java` | Placement with per-country settings | floor_price, max_price, device targeting, scheduling |
| `Instance.java` | Instance with device matching | placement_key, hb_status, manual eCPM per country, weight |
| `InstanceRule.java` | Rule with segment matching | brand/model/channel/OS/SDK version/IAP range/gender/age/tags |
| `InstanceRuleGroup.java` | Group within a rule | groupLevel, autoSwitch, abTest, per-variant priorities |
| `AdNetwork.java` | Network constants | 27+ network IDs (1=AdTiming, 3=Facebook, 5=Vungle, etc.) |
| `CountryCode.java` | ISO 3166-1 utility | alpha-2/3/numeric, "00" = ALL countries |

### Country Handling Patterns

1. ISO 3166-1 alpha-2 as primary format
2. `"00"` = ALL countries sentinel value
3. Per-country settings stored in separate tables (not inline)
4. **Universal fallback logic**: try specific country first, fall back to "00"

This pattern appears in:
- `Placement.isAllowPeriod(country)` — check country, fall back to ALL
- `Placement.getFloorAndMaxPrice(country)` — check country floor price, fall back to default
- `Instance.getInstanceManualEcpm(country)` — check country eCPM, fall back to "00"

### Fields Admedi Might Be Missing

Based on OM's more complete model:

**For `App`**: `platform` enum, `bundle_id`, `status` lifecycle, `coppa` flag

**For `Placement`**: `floor_price` (per-country), `max_price` (per-country), `frequency_cap/unit/interval`, `preload_timeout`, `inventory_count/interval`, `batch_size`, device targeting (OS/brand/model)

**For `Instance`**: `placement_key` (network's own ID), `hb_status` (bidding flag), `manual_ecpm` (per-country), `weight` (for weighted sorting), `frequency_cap/unit/interval`

**For `Group`/`WaterfallConfig`**: `sort_type` (auto/manual/weight), `group_level` (tier level), `ab_test_active` flag, `ab_test_id`

**For `TierTemplate`**: Sentinel value for "all countries" (OM uses "00")

---

## 8. AdMob API Samples

**Path**: `references/admob-api-samples/`
**Relevance**: Low for Core milestone; future reference for AdMob adapter.

### Python Samples

| File | Purpose |
|------|---------|
| `admob_utils.py` | Google OAuth2 auth helper |
| `list_accounts.py` | List publisher accounts |
| `list_apps.py` | List apps with pagination |
| `list_ad_units.py` | List ad units with pagination |
| `generate_network_report.py` | Network-level reporting |
| `generate_mediation_report.py` | Mediation-level reporting |

### OAuth2 Flow (vs ironSource)

| Aspect | ironSource/LevelPlay | AdMob |
|--------|---------------------|-------|
| Auth model | Machine-to-machine (key + token) | OAuth2 user consent |
| Token storage | In-memory (JWT cached) | Persistent file (pickle) |
| API client | Raw HTTP (httpx) | Google discovery client |
| Initial setup | Copy 2 values | Register GCP project, consent screen, download JSON |
| Scopes | Implicit | Explicit (`admob.readonly`, etc.) |

### Critical Finding: No Mediation Management API

The AdMob API is **reporting and inventory read-only**. There is no:
- Mediation group CRUD
- Waterfall/priority configuration
- Instance management

This means a future Admedi AdMob adapter would be **read/audit only**, not a sync adapter, unless Google extends the API.

### Adapter Interface Implication

Admedi's `MediationAdapter` should distinguish between:
- **Full CRUD adapters** (LevelPlay, potentially MAX)
- **Read-only adapters** (AdMob)

Consider capability flags or `NotImplementedError` for unsupported operations.

---

## Cross-Repository Patterns

### Pattern 1: Auth Token Caching

All repos that handle auth use the same fundamental pattern:
1. Check cached token validity (expiry check)
2. If expired, refresh
3. Cache the new token

**ironSource**: JWT `exp` claim (milliseconds), in-memory cache
**AdMob**: Google OAuth2 with persistent pickle file
**Google MCP repos**: Lazy init, request-scoped access tokens

**Admedi should**: Use PyJWT for decoding, add safety margin, support both in-memory (LevelPlay) and persistent (future AdMob) token storage.

### Pattern 2: Diff/Reconciliation

Three repos implement config reconciliation:

| Repo | Approach | Granularity |
|------|----------|-------------|
| SotiAds | `listChanges(source, target, comparator)` → `{toAdd, toUpdate, toRemove}` | Entity-level |
| OM-Server | Rule/segment matching → waterfall construction | Runtime per-request |
| ironSource libs | No diffing (raw API calls only) | N/A |

**Admedi should**: Adopt SotiAds' generic diff pattern, enhance with field-level diffs and human-readable summaries.

### Pattern 3: Country/Region Handling

| Repo | Approach |
|------|----------|
| OM | Separate country tables, "00" = ALL, alpha-2 primary, fallback logic |
| SotiAds | No country support |
| ironSource | `groupCountries` as array on mediation groups |

**Admedi should**: Use ISO 3166-1 alpha-2, define a sentinel for "all countries", implement specific-country-then-fallback resolution.

### Pattern 4: MCP Server Architecture

Both Google MCP repos agree on:
1. **Coordinator singleton** — `FastMCP` instance in dedicated module
2. **Side-effect registration** — import tool modules for decorator side effects
3. **Tools for actions, Resources for reference data**
4. **Lazy credential initialization** — not at import time

**Admedi should**: Follow the GMS repo's pattern (standalone `fastmcp`, `ToolError`, `output_schema`, dual transport).

### Pattern 5: Error Handling Spectrum

| Repo | Approach |
|------|----------|
| ironSource Python | Generic `Exception` everywhere |
| ironSource JS | Same — generic exceptions |
| SotiAds | Per-operation try/catch, no retry |
| Google MCP Official | Raw propagation (no handling) |
| Google MCP GMS | `ToolError` with masked details |

**Admedi should**: Define typed exceptions (`AuthError`, `RateLimitError`, `ApiError`, `ValidationError`), use `ToolError` in MCP layer, implement retry with exponential backoff.

---

## Entity Mapping Across Repos

| Admedi Entity | ironSource Python | ironSource JS | SotiAds | OpenMediation | AdMob |
|----------------|-------------------|---------------|---------|---------------|-------|
| **App** | `get_apps()` response | Same | YAML `apps` key | `om_publisher_app` | `list_apps` response |
| **TierTemplate** | `groupCountries` field | Same | `ecpmFloors` (no geo) | `om_placement_rule_segment` | N/A |
| **Group** | Mediation group methods | Same | Mediation group sync | `om_placement_rule` | N/A |
| **WaterfallConfig** | `MediationGroupPriority` | `MediationGroupPriority` | Ad source allocations | `om_placement_rule_instance` | N/A |
| **Instance** | `InstanceConfig` (42 classes) | `InstanceConfig` (26 classes) | Ad source configs | `om_instance` | N/A |
| **Placement** | `Placement` + `Capping` + `Pacing` | Same model | Placement-level config | `om_placement` | `list_ad_units` |
| **SyncLog** | N/A | N/A | Console output only | N/A | N/A |
| **ConfigSnapshot** | N/A | N/A | N/A | N/A | N/A |
| **Credential** | `set_credentials()` | Same | Browser cookies | `om_adnetwork_app` | OAuth2 pickle |

`SyncLog` and `ConfigSnapshot` are Admedi-specific — no reference repo has equivalents because none operate as a config management layer.

---

## Per-Task Reference Guide

### Task: Project Foundation

| What to Study | Repo | Files |
|---------------|------|-------|
| Enum values (Networks, AdUnits, TierType, etc.) | ironSource Python | `monetize_api/__init__.py` |
| Complete network list (29 networks) | ironSource JS | `models/instance_config.ts` |
| Waterfall data model (3 tiers + bidders) | ironSource Python | `monetize_api/mediation_group_priority.py` |
| Instance config fields per network | ironSource Python | `monetize_api/instance_config.py` |
| Placement model (capping, pacing) | ironSource Python | `monetize_api/placement_config.py` |
| Multi-network entity relationships | OpenMediation | `om-init.sql.gz` schema |
| Additional model fields to consider | OM-Server | Java DTOs (see Section 7) |

### Task: LevelPlay Authentication

| What to Study | Repo | Files |
|---------------|------|-------|
| JWT caching and exp claim checking | ironSource Python | `base_api.py` |
| Auth endpoint and header format | ironSource Python | `utils.py` (`BARRIER_AUTH_URL`) |
| httpx.AsyncClient patterns | ironSource Python | `utils.py` (`execute_request`) |
| Credential invalidation on change | ironSource Python | `base_api.py` (`set_credentials`) |
| Token caching (same flow, TypeScript) | ironSource JS | `base_api.ts` |

### Task: LevelPlay Read & Write

| What to Study | Repo | Files |
|---------------|------|-------|
| All 18 endpoint implementations | ironSource Python | `monetize_api/monetize_api.py` |
| Request body construction (instances) | ironSource Python | `monetize_api/monetize_api.py` |
| Pagination patterns | ironSource Python | `utils.py` (`execute_request_with_pagination`) |
| API version confirmation (v3 instances) | ironSource JS | `monetize_api.ts` |
| Delete placement body format | ironSource JS | `monetize_api.ts` |
| Rate limit awareness | (none — build from scratch) | — |

### Task: ConfigEngine Pipeline

| What to Study | Repo | Files |
|---------------|------|-------|
| `listChanges()` diff pattern | SotiAds | `src/base.ts` |
| YAML config schema design | SotiAds | `config.yml`, `src/read.ts` |
| Default/override hierarchy | SotiAds | `src/index.ts` (line 40-42) |
| Country fallback resolution | OM-Server | `Placement.java`, `Instance.java` |
| Waterfall sort type handling | OM-Server | `InstanceRuleGroup.java` |
| A/B test detection patterns | OM-Server | `InstanceRule.java` |

### Task: CLI, MCP Server & Storage

| What to Study | Repo | Files |
|---------------|------|-------|
| Coordinator singleton pattern | Google MCP GMS | `ads_mcp/coordinator.py` |
| Tool definitions with `output_schema` | Google MCP GMS | `ads_mcp/tools/api.py` |
| `ToolError` error handling | Google MCP GMS | `ads_mcp/tools/api.py`, `ads_mcp/tools/docs.py` |
| MCP Resources for reference data | Google MCP GMS | `ads_mcp/tools/docs.py` |
| Dual transport (stdio + HTTP) | Google MCP GMS | `ads_mcp/server.py`, `ads_mcp/stdio.py` |
| Dynamic description embedding | Google MCP Official | `ads_mcp/tools/search.py` |
| Fully mocked test patterns | Google MCP GMS | `tests/` |
| CLI design (sync + list commands) | SotiAds | `src/index.ts` |

### Task: Portfolio Dogfood

| What to Study | Repo | Files |
|---------------|------|-------|
| Per-app error isolation | SotiAds | `src/index.ts` (try/catch per format) |
| Tagged per-app logging | SotiAds | `consola.withTag(app.appId)` |
| Multi-app sequential processing | SotiAds | `src/index.ts` main loop |

---

## Key Findings

### 1. Admedi Is First-of-Its-Kind

No open-source tool wraps LevelPlay CRUD APIs (Groups v4, Instances, Placements) in any language. The official ironSource libs only cover reporting. Same gap exists for MAX. Admedi enters uncharted territory.

### 2. SotiAds Validates the Concept

SotiAds proves YAML-config-to-API-sync works for ad mediation. But it uses fragile undocumented RPCs, has no dry-run mode, no geo tiers, and no adapter abstraction. Admedi addresses all of these.

### 3. AdMob Has No Public Mediation Management API

Future AdMob adapter is limited to read/audit operations unless Google extends the API. The `MediationAdapter` interface should support capability-based feature detection.

### 4. The GMS MCP Repo Is the Gold Standard

For Admedi's FastMCP server, follow the GMS repo's patterns: standalone `fastmcp`, coordinator singleton, `ToolError`, `output_schema`, lazy credentials, fully mocked tests.

### 5. API Version Discrepancies Need Live Testing

Both ironSource libs use `/instances/v3` while current Unity docs show `/instances/v1`. The libs' versions are known to work. Admedi should test both but expect v3 to be correct.

### 6. OpenMediation's Country Fallback Pattern Is Universal

The "try specific country, fall back to ALL" pattern appears throughout OM's codebase. Admedi should adopt this as a core pattern in TierTemplate resolution.

### 7. 42 Instance Config Classes Is an Anti-Pattern

The ironSource Python lib's approach of one class per network (1549 lines of boilerplate) should be replaced with a data-driven network schema registry.

### 8. No Reference Repo Has Config Snapshots or Sync Logs

`SyncLog` and `ConfigSnapshot` are Admedi-specific innovations. No reference repo operates as a config management layer — they are either raw API wrappers, mediation platforms, or reporting tools. This is core differentiating functionality.
