# LevelPlay Monetization API Reference

**Version:** 1.1
**Date:** February 25, 2026
**Purpose:** Complete API surface documentation for building a LevelPlay mediation management CLI tool.

---

## Base URL

```
https://platform.ironsrc.com
```

All endpoints are prefixed with `/partners/publisher/` (legacy) or `/levelPlay/` (newer APIs).

---

## Authentication

LevelPlay uses **OAuth 2.0 Bearer Token** authentication.

**Credentials required:**
- `secretKey` — unique per ironSource account (found in dashboard account settings)
- `refreshToken` — used to generate bearer tokens

**Token endpoint:**

```
GET /partners/publisher/auth
```

**Headers:**
```
secretkey: {your_secret_key}
refreshToken: {your_refresh_token}
```

**Response:** Bearer token string (valid for **60 minutes**)

**Usage in all subsequent requests:**
```
Authorization: Bearer {token}
```

> **Note:** Some older endpoints also support Basic HTTP Auth (`base64(username:secretKey)`) but Bearer is the standard going forward.

---

## 1. Application API

Manage apps registered in your LevelPlay account.

### GET — List Applications

```
GET /partners/publisher/applications/v6
```

**Auth:** Bearer token
**Rate limit:** Standard

**Optional query parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| (none required) | — | Returns all apps by default |

**Response:**
```json
[
  {
    "appKey": "6be7c3bd",
    "appName": "Shelf Sort - Organize & Match",
    "appStatus": "active",
    "platform": "Android",
    "bundleId": "com.example.shelfsort",
    "taxonomy": "Casual",
    "creationDate": "2025-06-15",
    "icon": "https://...",
    "adUnits": {
      "rewardedVideo": {
        "activeNetworks": ["ironSource", "UnityAds", "AppLovin"]
      },
      "interstitial": {
        "activeNetworks": ["ironSource", "UnityAds", "AppLovin", "Pangle"]
      },
      "banner": {
        "activeNetworks": ["ironSource", "UnityAds"]
      }
    },
    "networkReportingApi": true,
    "bundleRefId": "...",
    "coppa": false,
    "ccpa": "do_not_sell"
  }
]
```

### POST — Add Application

```
POST /partners/publisher/applications/v6
```

Creates a new app in your LevelPlay account.

---

## 2. Groups API v4 ★ (Primary target for tier management)

Manage mediation groups — this is the core API for country tier configuration.

### Base Endpoint

```
https://platform.ironsrc.com/levelPlay/groups/v4/{appKey}
```

**Auth:** Bearer token
**Rate limit:** 4,000 requests per 30 minutes
**Scope:** 1 app per call

### GET — Retrieve Groups

```
GET /levelPlay/groups/v4/{appKey}
```

**Response:**
```json
[
  {
    "groupId": 12673,
    "groupName": "Tier 2",
    "mediationAdUnitId": "fgx25t56dq201bd2",
    "mediationAdUnitName": "interstitial-1",
    "adFormat": "interstitial",
    "abTest": "A",
    "countries": ["AU", "DE", "GB", "CA", "JP", "NZ", "KR", "TW"],
    "position": 2,
    "segments": [],
    "floorPrice": 0.3,
    "instances": [
      {
        "id": 3681,
        "name": "Default",
        "networkName": "ironSource",
        "isBidder": true
      },
      {
        "id": 2,
        "name": "Default",
        "networkName": "unityAds",
        "isBidder": false,
        "groupRate": 5,
        "countriesRate": [
          {
            "countryCode": "AU",
            "rate": 3.5
          },
          {
            "countryCode": "JP",
            "rate": 6.0
          }
        ]
      }
    ]
  }
]
```

**Key response fields:**

| Field | Type | Description |
|-------|------|-------------|
| `groupId` | int | Unique group identifier |
| `groupName` | string | Display name (e.g., "Tier 2") |
| `mediationAdUnitId` | string | Ad unit this group belongs to |
| `mediationAdUnitName` | string | Ad unit display name |
| `adFormat` | string | `interstitial`, `rewardedVideo`, `banner` |
| `abTest` | string | A/B test variant (`"A"`, `"B"`, or `"N/A"`) |
| `countries` | string[] | ISO 3166-1 alpha-2 country codes |
| `position` | int | Group priority (1 = highest, overrides lower) |
| `segments` | array | User segment filters |
| `floorPrice` | float | Minimum bid floor |
| `instances` | array | Ad network instances in this group |
| `instances[].id` | int | Instance ID |
| `instances[].networkName` | string | Ad network name |
| `instances[].isBidder` | bool | Whether instance participates in bidding |
| `instances[].groupRate` | float | Default eCPM rate for this group |
| `instances[].countriesRate` | array | Country-specific rate overrides |

### POST — Create Groups

```
POST /levelPlay/groups/v4/{appKey}
```

**Supports creating multiple groups in a single call.**

The `instances` parameter determines which instances to include/exclude. Instance IDs come from the Instances API.

If `adSourcePriority` is omitted, all active networks are placed in tier 2 with `sortByCPM` type.

### PUT — Update Groups

```
PUT /levelPlay/groups/v4/{appKey}
```

**Supports updating multiple groups in a single call.**

Optional fields — only include what you want to change:

| Field | Type | Description |
|-------|------|-------------|
| `groupName` | string | New group name |
| `groupPosition` | int | New priority position |
| `groupCountries` | string[] | Replace country list |
| `groupSegments` | array | Replace segments |
| `adSourcePriority` | object | Replace waterfall config |
| `floorPrice` | float | New floor price |
| `capping` | object | Frequency capping config |

> **Important:** Groups have a hierarchy — group at position 1 overrides group at position 2 for overlapping countries. If a country appears in multiple groups, only the highest-priority group applies.

> **Restriction:** The Mediation Management API will **not work** if you have an active A/B test running on the app.

---

## 3. Mediation Management API v2 (Legacy, more detailed waterfall control)

Full waterfall configuration with tier-level control.

### GET — Retrieve Mediation Config

```
GET /partners/publisher/mediation/management/v2?appKey={appKey}&adUnit={adUnit}
```

### PUT — Update Mediation Config

```
PUT /partners/publisher/mediation/management/v2
```

**Full request body example:**
```json
{
  "appKey": "ab1234",
  "adUnit": "interstitial",
  "groupName": "Tier 2",
  "groupCountries": ["AU", "DE", "GB", "CA", "JP", "NZ", "KR", "TW"],
  "adSourcePriority": {
    "bidding": {
      "tierType": "bidding",
      "instances": [
        {
          "providerName": "Pangle",
          "instanceId": 212121,
          "capping": {
            "value": 1,
            "interval": "session"
          }
        }
      ]
    },
    "tier1": {
      "tierType": "manual",
      "instances": [
        {
          "instanceName": "Default",
          "providerName": "AppLovin",
          "instanceId": 0,
          "rate": 8.0
        }
      ]
    },
    "tier2": {
      "tierType": "sortByCpm",
      "instances": [
        {
          "instanceName": "Default",
          "providerName": "UnityAds",
          "instanceId": 0,
          "rate": 4.5
        },
        {
          "providerName": "TapJoy",
          "instanceId": 54321,
          "rate": 3.0
        }
      ]
    },
    "tier3": {
      "tierType": "sortByCpm",
      "instances": [
        {
          "providerName": "InMobi",
          "instanceId": 0,
          "rate": 1.0
        }
      ]
    }
  }
}
```

**Tier types:**

| tierType | Behavior |
|----------|----------|
| `bidding` | Real-time bidding auction |
| `manual` | Fixed priority order (order in array = priority) |
| `sortByCpm` | Auto-sorted by eCPM performance |
| `optimized` | ironSource auto-optimization algorithm |

**Fields in PUT (all optional — omit to keep existing):**

| Field | Description |
|-------|-------------|
| `groupName` | Rename the group |
| `groupPosition` | Reorder priority |
| `groupSegments` | User segment targeting |
| `groupCountries` | Country list (replaces existing) |
| `adSourcePriority` | Full waterfall replacement |
| `capping` | Frequency capping |
| `tierType` | Default tier behavior |
| `rate` | Instance-level eCPM rate |

> **Default behavior:** If `tierType` is omitted, all tiers default to `sortByCPM`. Bidding sources automatically get `tierType: "bidding"`.

---

## 4. Instances API v1

Manage ad network instances (bidding and non-bidding).

### Base Endpoint

```
https://platform.ironsrc.com/partners/publisher/instances/v1?appKey={appKey}
```

**Auth:** Bearer token
**Rate limit:** 8,000 requests per 30 minutes
**Scope:** 1 app per call
**Note:** Bidding instances supported on GET only.

### GET — List Instances

```
GET /partners/publisher/instances/v1?appKey={appKey}
```

Returns all instances (bidding and non-bidding) for the app.

### POST — Create Instances

```
POST /partners/publisher/instances/v1
```

**Supports creating multiple instances in a single call.**

```json
{
  "appKey": "ab1234",
  "instances": [
    {
      "instanceName": "Pangle US High",
      "adUnit": "interstitial",
      "providerName": "Pangle",
      "isLive": true,
      "isOptimized": true,
      "globalPricing": 5.0,
      "countriesPricing": [
        { "country": "US", "eCPM": 12.0 },
        { "country": "JP", "eCPM": 8.0 }
      ]
    }
  ]
}
```

### PUT — Update Instances

```
PUT /partners/publisher/instances/v1
```

```json
{
  "appKey": "ab1234",
  "instances": [
    {
      "instanceId": 12345,
      "adUnit": "interstitial",
      "instanceName": "Pangle US High v2",
      "isLive": true,
      "globalPricing": 6.0,
      "countriesPricing": [
        { "country": "US", "eCPM": 14.0 }
      ]
    }
  ]
}
```

### DELETE — Delete Instances

```
DELETE /partners/publisher/instances/v1
```

Supports deleting multiple instances in a single call.

> **Error handling:** If any single instance in a batch request fails, the **entire request is rejected** with HTTP 400 + error array. Design for atomic operations or individual calls for fault tolerance.

---

## 5. Placements API v1

Manage ad placements (where/how ads are served in-app).

### Base Endpoint

```
https://platform.ironsrc.com/partners/publisher/placements/v1
```

**Auth:** Bearer token

### GET — List Placements

```
GET /partners/publisher/placements/v1?appKey={appKey}
```

**Response:**
```json
{
  "appKey": "ab1234",
  "placements": [
    {
      "adUnit": "interstitial",
      "id": 5678,
      "name": "LevelComplete",
      "adDelivery": 1,
      "capping": {
        "enabled": true,
        "amount": 5,
        "interval": "day"
      },
      "pacing": {
        "enabled": true,
        "seconds": 30
      }
    }
  ]
}
```

### POST — Create Placements

```
POST /partners/publisher/placements/v1
```

Supports multi-placement creation in a single call. Each placement is scoped to 1 ad unit.

**Supported ad units:** `rewardedVideo`, `interstitial`, `banner`

**Constraint:** Placement name must be unique per ad unit type.

### PUT — Update Placements

```
PUT /partners/publisher/placements/v1
```

Requires `id` field (from GET response). Supports capping/pacing updates.

**Constraint:** Default placement `adDelivery` cannot be toggled off.

### DELETE — Archive Placements

```
DELETE /partners/publisher/placements/v1
```

Requires `id` field.

---

## 6. Reporting API v1

Pull monetization performance data.

### Endpoint

```
GET /levelPlay/reporting/v1
```

**Auth:** Bearer token
**Rate limit:** 8,000 requests per hour

### Query Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `startDate` | string | Yes | `YYYY-MM-DD` |
| `endDate` | string | Yes | `YYYY-MM-DD` |
| `metrics` | string | Yes | Comma-separated (see below) |
| `breakdowns` | string | No | Comma-separated (see below) |

**Filters (all optional):**

| Filter | Description |
|--------|-------------|
| `app` | Filter by app key |
| `platform` | `Android`, `iOS` |
| `adNetwork` | Network name |
| `isBidder` | `true` / `false` |
| `adFormat` | `interstitial`, `rewardedVideo`, `banner` |
| `instance` | Instance ID |
| `country` | ISO country code |
| `mediationGroup` | Group ID |
| `mediationAdUnit` | Ad unit ID |
| `segment` | Segment name |
| `placement` | Placement name |
| `osVersion` | OS version string |
| `sdkVersion` | SDK version |
| `appVersion` | App version |

**Available metrics:** `revenue`, `eCPM`, `impressions`, `activeUsers`, `engagedUsers`, `engagedSessions`, `impressionsPerEngagedSession`, `revenuePerActiveUser`, `fillRate`, `attempts`, `responses`

**Available breakdowns:** `date`, `app`, `platform`, `adNetwork`, `isBidder`, `adFormat`, `instance`, `country`, `mediationGroup`, `mediationAdUnit`, `segment`, `placement`, `osVersion`, `sdkVersion`, `appVersion`

---

## 7. Impression Level Revenue (ILR) Server-Side API

Granular per-impression revenue data.

### Characteristics

- **Data delay:** 1 day
- **Scope:** 1 app key + 1 date per request
- **Delivery:** Returns a download URL; file available for **1 hour**
- **Revenue calculation:** Based on eCPM per impression (app × network × country × instance)

### Two APIs available:

| API | Data Level | Use Case |
|-----|-----------|----------|
| Device Level | Per-device aggregated | ARPDAU analysis, user segmentation |
| Impression Level | Per-impression | Detailed auction/waterfall analysis |

---

## Rate Limits Summary

| API | Limit |
|-----|-------|
| Groups API v4 | 4,000 / 30 min |
| Instances API v1 | 8,000 / 30 min |
| Reporting API v1 | 8,000 / 1 hour |
| Application API | Standard |
| Placements API | Standard |
| Mediation Mgmt v2 | Standard |

All APIs return **HTTP 429** when rate limited.

---

## Error Handling

All APIs return standard HTTP codes:

| Code | Meaning |
|------|---------|
| 200 | Success |
| 400 | Bad request (check error array for details) |
| 401 | Auth failure (token expired or invalid) |
| 404 | Resource not found (bad appKey, groupId, etc.) |
| 429 | Rate limited |
| 500 | Server error |

**Batch operation behavior (Instances API):** If any item in a batch fails, the **entire batch is rejected**. Error response includes per-item error messages.

---

## Known Limitations / Dashboard-Only Features

| Feature | API Support |
|---------|-------------|
| A/B test creation/management | Dashboard only |
| Segment creation | Dashboard only (assignment to groups via API works) |
| Ad network credential setup | Dashboard only (initial onboarding) |
| Active A/B test + Mediation API | **Blocked** — API calls fail if A/B test is running |

---

## Country Code Reference (ISO 3166-1 alpha-2)

For our tier configuration:

| Tier | Countries | Codes |
|------|-----------|-------|
| Tier 1 | United States | `US` |
| Tier 2 | Australia, Canada, Germany, Japan, New Zealand, South Korea, Taiwan, United Kingdom | `AU`, `CA`, `DE`, `JP`, `NZ`, `KR`, `TW`, `GB` |
| Tier 3 | France, Netherlands | `FR`, `NL` |
| All Countries | Catch-all | (default group) |

---

## Existing Open-Source Base: ironSource/mobile-api-lib-python

**Repo:** [github.com/ironSource/mobile-api-lib-python](https://github.com/ironSource/mobile-api-lib-python)
**Package:** `ironsrc_mobile_api` (PyPI)
**License:** Apache-2.0
**Last commit:** December 13, 2022 (v1.1.0)
**Language:** Python 3.8+

### Package Structure

```
ironsource_api/
├── __init__.py
├── base_api.py                      # Auth (Bearer + Basic), credential management
├── ironsource_api.py                # Entry point — IronSourceAPI class
├── utils.py                         # HTTP client (httpx async), gzip, pagination, streaming
├── monetize_api/
│   ├── __init__.py                  # Enums: AdUnits, Networks, Metrics, Breakdowns, Platform
│   ├── monetize_api.py              # 18 endpoint methods (see coverage below)
│   ├── instance_config.py           # InstanceConfig dataclass for typed payloads
│   ├── mediation_group_priority.py  # MediationGroupPriority + TierType classes
│   └── placement_config.py          # Placement dataclass for typed payloads
└── promote_api/
    ├── __init__.py
    └── promote_api.py               # UA/Promote campaign APIs
```

### Already Implemented (18 endpoints)

| Category | Methods | Endpoint Used |
|----------|---------|---------------|
| **Applications** | `get_apps()`, `add_app()`, `add_temporary_app()` | `/applications/v6` |
| **Mediation Groups** | `get_mediation_groups()`, `create_mediation_group()`, `update_mediation_group()`, `delete_mediation_group()` | `/mediation/management/v2` |
| **Instances** | `get_instances()`, `add_instances()`, `update_instances()`, `delete_instance()` | `/instances/v3` |
| **Placements** | `get_placements()`, `add_placements()`, `update_placements()`, `delete_placements()` | `/placements/v1` |
| **Reporting** | `get_monetization_data()` | `/mediation/applications/v6/stats` |
| **ILR** | `get_user_ad_revenue()`, `get_impression_ad_revenue()` | `/userAdRevenue/v3`, `/adRevenueMeasurements/v3` |

### Existing URL Constants (from monetize_api.py)

```python
APP_API_URL = "https://platform.ironsrc.com/partners/publisher/applications/v6"
REPORT_URL = "https://platform.ironsrc.com/partners/publisher/mediation/applications/v6/stats"
UAR_URL = "https://platform.ironsrc.com/partners/userAdRevenue/v3"
ARM_URL = "https://platform.ironsrc.com/partners/adRevenueMeasurements/v3"
MEDIATION_GROUP_MGMT_URL = "https://platform.ironsrc.com/partners/publisher/mediation/management/v2"
INSTANCES_API_URL = "https://platform.ironsrc.com/partners/publisher/instances/v3"
PLACEMENTS_URL = "https://platform.ironsrc.com/partners/publisher/placements/v1"
```

### Auth Architecture (base_api.py)

```python
class BaseAPI:
    # Stores credentials as private class vars
    # Bearer token auto-refreshes: checks JWT exp claim, re-fetches if expired
    # Token lifetime: 60 minutes
    # Methods:
    #   set_credentials(user, token, secret)
    #   async get_bearer_auth() -> str        # Auto-caches + auto-refreshes
    #   get_basic_auth() -> str               # base64(user:secret)
```

### HTTP Client (utils.py)

```python
# Async via httpx.AsyncClient (60s timeout)
# Supports: gzip responses, streaming, pagination
# Key functions:
#   execute_request(method, url, is_gzip, **kwargs) -> ResponseInterface
#   execute_request_as_stream(url, is_gzip) -> bytes
#   execute_request_with_pagination(url, pipe_w, data_key, err_string, options, as_bytes)
#   get_bearer_auth(secret, token) -> str
#   get_basic_auth(username, secret) -> str
```

### Existing Data Models

**MediationGroupPriority** — typed waterfall config builder:
```python
# Builds adSourcePriority JSON with tierType support
# TierType enum: MANUAL, SORT_BY_CPM, OPTIMIZED, BIDDING
```

**InstanceConfig** — typed instance payload builder:
```python
# Fields: instanceName, adUnit, providerName, isLive, isOptimized,
#          globalPricing, countriesPricing
```

**Placement** — typed placement payload builder:
```python
# Fields: adUnit, name, adDelivery, capping, pacing
```

### What Needs to Be Added (Fork Scope)

| Gap | What's Missing | Action |
|-----|---------------|--------|
| **Groups API v4** | Newer endpoint (`/levelPlay/groups/v4/{appKey}`) not present | Add new methods alongside existing v2 |
| **Reporting API v1** | Newer endpoint (`/levelPlay/reporting/v1`) not present | Add new method alongside existing reporting |
| **Ad Units API** | Not implemented at all | New methods for ad unit CRUD |
| **CLI layer** | Library only — no command-line interface | Add CLI with config-file-driven bulk operations |
| **Config templating** | No concept of tier templates or multi-app fan-out | New module: define tiers once, apply to N app keys |
| **Endpoint versions** | Instances on v3 (latest docs show v1 path) | Verify current version, update if needed |
| **Python version** | Built for 3.8+ | May want to modernize to 3.10+ with match/case |

### Extension Strategy

**Phase 1 — Fork + Upgrade Endpoints**
- Fork repo, keep existing structure
- Add Groups API v4 methods to `monetize_api.py`
- Add Reporting API v1 alongside existing reporting
- Update any stale endpoint versions
- Keep backward compat with v2 mediation management

**Phase 2 — CLI Layer**
- New top-level module: `cli.py` (click or typer)
- Config file format (YAML): define tier templates with countries, floor prices
- Commands: `sync-tiers`, `get-config`, `diff-config`, `apply-config`
- Fan-out: iterate app keys × ad formats, apply template

**Phase 3 — Open Source**
- Clean up, add tests for new endpoints
- Publish as new PyPI package (e.g., `levelplay-cli` or `ironsource-manager`)
- README with usage examples

---

## Source Documentation

- [LevelPlay Monetization APIs Overview](https://developers.is.com/ironsource-mobile/general/levelplay-monetization-apis/)
- [Authentication](https://developers.is.com/ironsource-mobile/air/authentication-omer/)
- [Groups API v4](https://developers.is.com/ironsource-mobile/general/groups-api-v4/)
- [Mediation Management API v2](https://developers.is.com/ironsource-mobile/air/mediation-management-v2/)
- [Application API](https://developers.is.com/ironsource-mobile/air/application-api/)
- [Placements API](https://developers.is.com/ironsource-mobile/air/placements-api/)
- [Instances API (Unity Docs)](https://docs.unity.com/en-us/grow/is-ads/monetization/apis/ironsource-instances-api)
- [Reporting API](https://developers.is.com/ironsource-mobile/air/reporting/)
- [ILR Server-Side API](https://developers.is.com/ironsource-mobile/air/ad-revenue-measurements/)
- [ironSource/mobile-api-lib-python (GitHub)](https://github.com/ironSource/mobile-api-lib-python)
