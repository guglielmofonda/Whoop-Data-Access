# WHOOP Internal API — Reverse Engineering Guide

> **Sources**: Static analysis of `app.whoop.com` JavaScript bundle + live HAR captures
> **Date**: 2026-02-24
> **Athlete ID used in examples**: `YOUR_USER_ID`

---

## Table of Contents

1. [Overview](#overview)
2. [Authentication](#authentication)
3. [Base URL, Common Headers & Required Params](#base-url-common-headers--required-params)
4. [Endpoints](#endpoints)
   - [1. Cycles / Strain Data](#1-cycles--strain-data-primary-endpoint)
   - [2. User Bootstrap / Profile](#2-user-bootstrap--profile)
   - [3. Activities / Sports Catalog](#3-activities--sports-catalog)
   - [4. Membership Status](#4-membership-status)
   - [5. Sleep Events (Detailed Stage Timeline)](#5-sleep-events-detailed-stage-timeline)
5. [Date & Time Handling](#date--time-handling)
6. [Duration Codes → Date Ranges](#duration-codes--date-ranges)
7. [Pagination](#pagination)
8. [Response Schemas (Full)](#response-schemas-full)
   - [Cycles/Details Record](#cyclesdetails-record)
   - [Bootstrap Response](#bootstrap-response)
   - [Sports Catalog Entry](#sports-catalog-entry)
   - [Membership Response](#membership-response)
   - [Metrics Response](#metrics-response)
   - [Sleep Events Response](#sleep-events-response)
9. [cURL Examples](#curl-examples)
10. [Notes & Gotchas](#notes--gotchas)

---

## Overview

WHOOP's web app (`app.whoop.com`) communicates with a microservices API at `https://api.prod.whoop.com`. The app is AngularJS-based and uses OAuth2 Bearer token authentication. All endpoints accept/return JSON.

**These are internal/private API endpoints** — they are not part of WHOOP's public developer API (`developer.whoop.com`). They may change without notice.

**Architecture details:**
- Microservices: each domain has its own path prefix (`/users-service/`, `/activities-service/`, `/core-details-bff/`, etc.)
- OAuth2 Bearer token auth — no CSRF tokens needed
- Every request includes `apiVersion=7` as a query parameter
- Dates always in ISO 8601, anchored to the user's local timezone
- Response range strings use PostgreSQL interval notation: `['start','end')` (inclusive start, exclusive end)

---

## Authentication

### Get an Access Token (Login)

```
POST https://api.prod.whoop.com/api-server/oauth/token
Content-Type: application/json
```

**Request body:**
```json
{
  "grant_type": "password",
  "username": "your@email.com",
  "password": "yourpassword",
  "issueRefresh": true
}
```

**Response:**
```json
{
  "access_token": "eyJhbGci...",
  "refresh_token": "def50200...",
  "token_type": "Bearer",
  "expires_in": 3600,
  "user": {
    "id": YOUR_USER_ID,
    "membershipStatus": "active"
  }
}
```

Set `issueRefresh: true` to receive a `refresh_token` for long-lived sessions.

---

### Refresh an Expired Token

```
POST https://api.prod.whoop.com/api-server/oauth/token
Content-Type: application/json
```

```json
{
  "grant_type": "refresh_token",
  "refresh_token": "def50200...",
  "issueRefresh": false
}
```

Returns a new `access_token` (same response shape as login).

---

### Manual Token Extraction (Already Logged In)

If you're logged in to `app.whoop.com`, extract your token without a login call:

1. Open DevTools (`F12` / `Cmd+Option+I`) on any WHOOP page
2. **Application** tab → **Local Storage** → `https://app.whoop.com`
3. Copy these keys:
   - `security.accessToken` — the Bearer token for API calls
   - `security.refreshToken` — for renewal

**Or from a live request:**
1. DevTools → **Network** tab → click any request to `api.prod.whoop.com`
2. **Request Headers** → `authorization: bearer eyJhbGci...`

> **Note on HAR files**: Chrome strips `Authorization` and `Cookie` headers when saving HAR files (security feature). The header is confirmed present in the actual requests via JS source analysis.

---

## Base URL, Common Headers & Required Params

**Base URL:** `https://api.prod.whoop.com`

**Required header on all authenticated requests:**
```
Authorization: bearer <ACCESS_TOKEN>
```

> Use lowercase `bearer` — that's what the app sends.

**Required query parameter on every request:**
```
apiVersion=7
```

This must be appended to all requests. Omitting it may cause failures or different response shapes.

**For POST/PATCH requests, also add:**
```
Content-Type: application/json
```

---

## Endpoints

### 1. Cycles / Strain Data *(Primary Endpoint)*

Returns daily cycles with strain, recovery, sleep, and workout data. This powers all trend views on the profile page.

```
GET https://api.prod.whoop.com/core-details-bff/v0/cycles/details
```

**Query parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `apiVersion` | integer | Yes | Always `7` |
| `id` | integer | Yes | Your WHOOP user ID (e.g. `YOUR_USER_ID`) |
| `startTime` | ISO 8601 string | Yes | Start of range (user's local midnight, in UTC) |
| `endTime` | ISO 8601 string | Yes | End of range (user's local 23:59:59, in UTC) |
| `limit` | integer | No | Max number of cycles to return. App adds this automatically when the range > 10 days |

**Example — 2-week strain view ending 2026-02-17 (user in PST, UTC-8):**
```
GET https://api.prod.whoop.com/core-details-bff/v0/cycles/details
  ?apiVersion=7
  &id=YOUR_USER_ID
  &startTime=2026-02-03T08:00:00.000Z
  &endTime=2026-02-16T07:59:59.999Z
  &limit=12
```

> See [Date & Time Handling](#date--time-handling) for how to compute the correct UTC timestamps from a local date.

**What the app actually fetches for a 2-week view (two requests):**
1. The current/most-recent cycle(s): `startTime=<2 days ago>` → `endTime=<today end>` (no limit)
2. The historical range: `startTime=<start>` → `endTime=<yesterday end>` with `limit=<day count>`

---

### 2. User Bootstrap / Profile

Returns user account, profile, membership status, and optionally teams/profile.

```
GET https://api.prod.whoop.com/users-service/v2/bootstrap/
```

**Query parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `apiVersion` | integer | Yes | Always `7` |
| `accountType` | string | Yes | Always `"users"` |
| `id` | integer | Yes | Your WHOOP user ID |
| `include` | string | No | Repeatable — `include=profile` and/or `include=teams` to include extra objects |

**Examples:**

Minimal (account + user + bio_data + membership only):
```
GET https://api.prod.whoop.com/users-service/v2/bootstrap/
  ?apiVersion=7&accountType=users&id=YOUR_USER_ID
```

Full profile with teams:
```
GET https://api.prod.whoop.com/users-service/v2/bootstrap/
  ?apiVersion=7&accountType=users&id=YOUR_USER_ID&include=profile&include=teams
```

---

### 3. Activities / Sports Catalog

Returns the full list of all sport/activity types WHOOP supports. Use this to map `sport_id` integers in workout records to human-readable names.

```
GET https://api.prod.whoop.com/activities-service/v1/sports/history
```

**Query parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `apiVersion` | integer | Yes | Always `7` |

**Example:**
```
GET https://api.prod.whoop.com/activities-service/v1/sports/history?apiVersion=7
```

Returns an array of 200+ sport objects. No auth appears required for this endpoint (it's a static catalog), but send the Bearer token anyway.

---

### 5. Sleep Events (Detailed Stage Timeline)

Returns the full minute-by-minute sleep stage breakdown for a specific sleep session. This is what powers the sleep graph — it contains every LIGHT, SWS, REM, WAKE, LATENCY, and DISTURBANCE interval.

```
GET https://api.prod.whoop.com/sleep-service/v1/sleep-events
```

**Query parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `apiVersion` | integer | Yes | Always `7` |
| `activityId` | UUID string | Yes | The sleep session's `activity_id` (from cycles/details `sleeps[].activity_id`) |

**Example:**
```
GET https://api.prod.whoop.com/sleep-service/v1/sleep-events
  ?activityId=448541a7-7b7b-4701-87d0-da690fb0f09d
  &apiVersion=7
```

The `activityId` is found in `cycles/details` → `record.sleeps[0].activity_id`.

---

### 4. Membership Status

Returns billing, subscription, and membership details.

```
GET https://api.prod.whoop.com/membership
```

**Query parameters:** None required (no `apiVersion` needed on this endpoint based on HAR).

---

## Date & Time Handling

**Critical:** WHOOP's API boundaries are anchored to the user's **local timezone**, not UTC. The user's `timezone_offset` (e.g. `-0800` for PST) is stored in their profile.

**Day boundary conversion:**
- Start of day (local midnight) → add `abs(offset)` hours to get UTC
  - e.g., midnight PST (-0800) = `T08:00:00.000Z`
- End of day (local 23:59:59) → the next calendar day minus 1 second at UTC
  - e.g., 23:59:59 PST on Feb 16 = `2026-02-17T07:59:59.999Z`

**Real examples from HAR captures (user timezone: -0800):**

| Local date | Boundary | UTC value |
|---|---|---|
| 2026-02-03 | Start | `2026-02-03T08:00:00.000Z` |
| 2026-02-16 | End | `2026-02-16T07:59:59.999Z` |
| 2026-02-18 | Start | `2026-02-18T08:00:00.000Z` |
| 2026-02-18 | End (current) | `2026-02-18T07:59:59.999Z` |

**Python — compute range from URL params (timezone-aware):**
```python
from datetime import datetime, timezone, timedelta

def compute_whoop_range(until_date_str: str, duration_code: str, tz_offset_hours: int = -8):
    """
    until_date_str: "2026-02-17"  (the date in the URL)
    duration_code: "1w", "2w", "1m", "3m", "6m"
    tz_offset_hours: user's UTC offset, e.g. -8 for PST

    Returns startTime and endTime as ISO 8601 UTC strings.
    """
    days_map = {"1w": 7, "2w": 14, "1m": 30, "3m": 90, "6m": 180}
    days = days_map[duration_code]

    # Parse end date and compute local midnight → UTC
    until = datetime.strptime(until_date_str, "%Y-%m-%d")
    # End = local midnight of the day AFTER until_date, minus 1ms
    # i.e., 23:59:59.999 local = 23:59:59.999 + offset → UTC
    local_end = until + timedelta(days=1)          # next day's local midnight
    utc_end = local_end - timedelta(hours=tz_offset_hours)  # convert to UTC
    utc_end = utc_end - timedelta(milliseconds=1)  # back to 23:59:59.999

    # Start = end minus N days, at local midnight → UTC
    local_start = until - timedelta(days=days - 1)
    utc_start = local_start - timedelta(hours=tz_offset_hours)

    return {
        "startTime": utc_start.strftime("%Y-%m-%dT%H:%M:%S.000Z"),
        "endTime":   utc_end.strftime("%Y-%m-%dT%H:%M:%S.999Z"),
    }

# Example (matches HAR exactly):
# compute_whoop_range("2026-02-16", "2w", tz_offset_hours=-8)
# → {"startTime": "2026-02-03T08:00:00.000Z", "endTime": "2026-02-16T07:59:59.999Z"}
```

**JavaScript:**
```javascript
function computeWhoopRange(untilDateStr, durationCode, tzOffsetHours = -8) {
  const daysMap = { "1w": 7, "2w": 14, "1m": 30, "3m": 90, "6m": 180 };
  const days = daysMap[durationCode];
  const offsetMs = tzOffsetHours * 60 * 60 * 1000;

  // End: until date's local midnight → UTC, then back to 23:59:59.999
  const untilMidnightUTC = new Date(untilDateStr + "T00:00:00.000Z");
  const nextDayLocalMidnightUTC = new Date(untilMidnightUTC.getTime() + 24 * 60 * 60 * 1000);
  const endUTC = new Date(nextDayLocalMidnightUTC.getTime() - offsetMs - 1);

  // Start: (until - days + 1) days back, at local midnight → UTC
  const startLocalMidnight = new Date(untilMidnightUTC.getTime() - (days - 1) * 24 * 60 * 60 * 1000);
  const startUTC = new Date(startLocalMidnight.getTime() - offsetMs);

  return {
    startTime: startUTC.toISOString().replace(/\.\d+Z$/, ".000Z"),
    endTime:   endUTC.toISOString().replace(/\.\d+Z$/, ".999Z"),
  };
}
```

---

## Duration Codes → Date Ranges

| URL code | Days | App query behavior |
|----------|------|--------------------|
| `1w` | 7 | Single request |
| `2w` | 14 | Single request with `limit=12–14` |
| `1m` | 30 | Single request |
| `3m` | 90 | Split into 30-day chunks |
| `6m` | 180 | Split into 30-day chunks |

---

## Pagination

For the cycles endpoint with ranges > ~30 days, the app issues multiple parallel requests with chunked date ranges. Each chunk covers approximately 30 days. There is no offset/cursor-based pagination — each chunk has its own independent `startTime`/`endTime`.

**Example — 3-month fetch (90 days, split into 3 chunks):**
```
# Chunk 1 (oldest)
?startTime=2025-11-07T08:00:00.000Z&endTime=2025-12-09T07:59:59.999Z&limit=31

# Chunk 2
?startTime=2025-12-09T08:00:00.000Z&endTime=2026-01-08T07:59:59.999Z&limit=29

# Chunk 3 (most recent)
?startTime=2026-01-08T08:00:00.000Z&endTime=2026-02-03T07:59:59.999Z&limit=25
```

Merge and sort results by `cycle.during` after all requests complete.

---

## Response Schemas (Full)

### Cycles/Details Record

The top-level response is:
```json
{
  "records": [ <CycleRecord>, <CycleRecord>, ... ]
}
```

Each `CycleRecord` has this shape:

```json
{
  "cycle": {
    "id": 1313520632,
    "created_at": "2026-02-16T15:07:42.694+0000",
    "updated_at": "2026-02-17T15:23:23.174+0000",
    "user_id": YOUR_USER_ID,
    "during": "['2026-02-15T06:57:35.430Z','2026-02-16T07:09:07.160Z')",
    "days": "['2026-02-15','2026-02-16')",
    "timezone_offset": "-0800",
    "data_state": "complete",
    "scaled_strain": 13.400938,
    "day_strain": 0.00761,
    "day_kilojoules": 10370.79,
    "day_avg_heart_rate": 65,
    "day_max_heart_rate": 187,
    "sleep_need": null,
    "predicted_end": "2026-02-16T07:09:07.160+0000",
    "intensity_score": null
  },

  "sleeps": [
    {
      "activity_id": "some-uuid",
      "activity_version": 0,
      "user_id": YOUR_USER_ID,
      "created_at": "2026-02-16T15:07:42.416Z",
      "updated_at": "2026-02-16T15:07:42.416Z",
      "during": "['2026-02-15T06:57:35.430Z','2026-02-15T14:59:35.310Z')",
      "timezone_offset": "-08:00",
      "state": "complete",
      "source": "auto+user",
      "survey_response_id": null,
      "is_nap": false,
      "normal": true,
      "significant": true,
      "score": 92,
      "projected_score": 92.0,
      "sleep_consistency": 84.0,

      // Durations — all in MILLISECONDS
      "quality_duration": 27928890,
      "time_in_bed": 29233940.0,
      "light_sleep_duration": 12563150,
      "slow_wave_sleep_duration": 6512940,
      "rem_sleep_duration": 8852800,
      "wake_duration": 1305100,
      "arousal_time": 600000.0,
      "no_data_duration": 0,
      "latency": 0,
      "projected_sleep": 27928890.0,
      "credit_from_naps": 0.0,

      // Sleep debt — in MILLISECONDS
      "sleep_need": 30004136.0,
      "habitual_sleep_need": 28053705.0,
      "need_from_strain": 1950431.0,
      "debt_pre": 2075712.0,
      "debt_post": 5006822.0,

      // Vitals
      "respiratory_rate": 13.828125,
      "in_sleep_efficiency": 0.95037776,

      // Sleep cycles & disruptions
      "cycles_count": 7,
      "disturbances": 8,
      "total_wake_events": 11,

      // Optimal window (PostgreSQL range string)
      "optimal_sleep_times": "['2026-02-15T06:15:00.000Z','2026-02-15T14:55:00.000Z')",

      "algo_version": "9.1.8.2"
    }
  ],

  "recovery": {
    "activity_id": "some-uuid",
    "activity_version": 0,
    "user_id": YOUR_USER_ID,
    "created_at": "2026-02-16T15:07:42.416Z",
    "updated_at": "2026-02-16T15:07:42.416Z",
    "state": "complete",
    "responded": false,
    "survey_response_id": null,
    "calibrating": false,
    "normal": true,
    "recovery_score": 95,
    "resting_heart_rate": 46,
    "hrv_rmssd": 0.099190265,
    "hr_baseline": 48.0,
    "skin_temp_celsius": 33.199333,
    "spo2": 97.5,
    "prob_covid": 0.104,
    "rhr_component": 0.7980845,
    "hrv_component": 0.93978375,
    "history_size": 8.0,
    "recovery_rate": null,
    "algo_version": "mav.8.3.0"
  },

  "workouts": [
    {
      "activity_id": "d9df8189-fd7a-425b-a47c-466d6c7fdf31",
      "activity_version": 2,
      "during": "['2026-02-16T01:09:00.000Z','2026-02-16T02:05:59.720Z')",
      "timezone_offset": "-08:00",
      "source": "user",
      "survey_response_id": null,
      "sport_id": 45,

      "score": 12.696353,
      "intensity_score": 12.696353,
      "cumulative_workout_intensity": 15.182196,
      "raw_intensity_score": 0.00638894,
      "percent_recorded": 0.99978,
      "kilojoules": 1187.046,
      "max_heart_rate": 161,
      "average_heart_rate": 108,
      "total_steps": null,
      "gps_data": null,

      // Heart rate zones — durations in SECONDS (% of max HR)
      "zone_durations_v2": {
        "zone0_to50_duration": 1214.02,
        "zone50_to60_duration": 1829.96,
        "zone60_to70_duration": 284.0,
        "zone70_to80_duration": 91.0,
        "zone80_to90_duration": 0.0,
        "zone90_to100_duration": 0.0
      },
      // Legacy zone array: [zone0-50, zone50-60, zone60-70, zone70-80, zone80-90, zone90-100]
      "zone_durations": [1214.02, 1829.96, 284.0, 91.0, 0.0, 0.0],

      // MSK (musculoskeletal) score — present for strength activities, null for cardio-only
      "msk_score": {
        "raw_cardio_intensity_score": 0.001739,
        "scaled_cardio_intensity_score": 7.094,
        "raw_msk_intensity_score": 0.159138,
        "scaled_msk_intensity_score": 11.148,
        "cardio_strain_contribution_percent": 0.27226
      }
    }
  ],

  "v2_activities": [
    {
      "id": "d9df8189-fd7a-425b-a47c-466d6c7fdf31",
      "cycle_id": 1313520632,
      "user_id": YOUR_USER_ID,
      "created_at": "2026-02-16T02:21:16.167+0000",
      "updated_at": "2026-02-16T04:12:33.681+0000",
      "version": 2,
      "during": "['2026-02-16T01:09:00.000Z','2026-02-16T02:05:59.720Z')",
      "timezone": "America/Los_Angeles",
      "timezone_offset": null,
      "timezone_offset_from_model": "-08:00",
      "source": "user",
      "source_id": null,
      "activity_v1_id": null,
      "score_state": "complete",
      "score_type": "CARDIO",
      "type": "weightlifting",
      "translated_type": "Weightlifting"
    }
  ]
}
```

**Field reference for key values:**

| Field | Unit | Notes |
|---|---|---|
| `scaled_strain` | 0–21 | Full-day strain score |
| `day_kilojoules` | kJ | Total energy expenditure |
| `sleep.score` | 0–100 | Sleep performance % |
| `sleep.*_duration` | milliseconds | All sleep duration fields |
| `sleep.respiratory_rate` | breaths/min | |
| `recovery.recovery_score` | 0–100 | Recovery % |
| `recovery.hrv_rmssd` | seconds | Convert to ms: multiply by 1000 |
| `recovery.resting_heart_rate` | bpm | |
| `recovery.skin_temp_celsius` | °C | |
| `recovery.spo2` | % | Blood oxygen saturation |
| `workout.score` | 0–21 | Workout strain score |
| `workout.kilojoules` | kJ | Workout energy expenditure |
| `workout.zone_durations_v2.*` | seconds | Time in each HR zone |
| `during` / `days` / `optimal_sleep_times` | — | PostgreSQL range `[start,end)` |

**Parsing the range string:**
```python
import re

def parse_range(range_str):
    # e.g. "['2026-02-16T07:31:39.270Z','2026-02-17T07:28:41.430Z')"
    parts = re.findall(r"[\d\-T:.Z]+", range_str)
    return {"start": parts[0], "end": parts[1]}
```

---

### Bootstrap Response

```json
{
  "account": {
    "id": YOUR_ACCOUNT_ID,
    "username": "yourusername",
    "email": "user@email.com",
    "type": "user",
    "can_upload_data": true,
    "deidentified": false,
    "concealed": false,
    "disabled": false,
    "tos_accepted": "2025-03-27T00:00:00.000Z",
    "created_at": "2025-03-27T22:53:48.818Z",
    "updated_at": "2026-02-24T21:39:05.459Z",
    "user_id": YOUR_USER_ID,
    "staff_id": null
  },
  "user": {
    "id": YOUR_USER_ID,
    "first_name": "FirstName",
    "last_name": "Name",
    "country": "US",
    "city": "San Francisco",
    "admin_division": "YY",
    "avatar_url": "https://s3-us-west-2.amazonaws.com/avatars.whoop.com/uploads/...",
    "created_at": "2025-03-27T22:53:48.815Z",
    "updated_at": "2025-10-14T00:09:17.215Z"
  },
  "staff": null,
  "teams": [],
  "profile": {
    "user_id": YOUR_USER_ID,
    "bio_data_id": 661150185,
    "height": 1.80,
    "weight": 80.00,
    "gender": "male",
    "unit_system": "metric",
    "fitness_level": "recreational_enthusiast",
    "birthday": "1990-01-01T00:00:00.000Z",
    "timezone_offset": "-0800",
    "physiological_baseline": null,
    "created_at": "2025-03-27T22:54:18.988Z",
    "updated_at": "2026-02-24T16:14:26.077Z"
  },
  "membership": {
    "status": "trialing",
    "in_effect": true
  },
  "bio_data": {
    "max_heart_rate": 197,
    "min_heart_rate": 44,
    "resting_heart_rate": 49,
    "recovery_count": 334
  }
}
```

> `profile` and `teams` only appear when you include `include=profile&include=teams` in the request.
> Without those params, `profile` still appears but `teams` will be an empty array.

---

### Sports Catalog Entry

```json
{
  "id": 45,
  "name": "Weightlifting",
  "category": "cardiovascular",
  "activity_type_internal_name": "weightlifting",
  "icon_url": "https://s3-us-west-2.amazonaws.com/icons.whoop.com/mobile/activities/weightlifting.png",
  "icon_svg_url": "https://s3-us-west-2.amazonaws.com/icons.whoop.com/mobile/activities/weightlifting.svg",
  "has_gps": false,
  "is_current": true,
  "has_survey": false,
  "created_at": "2014-02-11T10:07:16.000+0000",
  "updated_at": "2022-06-03T20:06:45.412+0000"
}
```

Returns 200+ entries. Use `sport_id` from workout records to look up the name here.

---

### Membership Response

```json
{
  "userId": YOUR_USER_ID,
  "membershipStatus": "trialing",
  "subscriptionType": "annual",
  "trialStart": "YYYY-MM-DDTHH:MM:SS.000Z",
  "trialDays": 365,
  "expirationDate": null,
  "canceledAt": null,
  "cancelAtPeriodEnd": false,
  "nextBillDate": "YYYY-MM-DDTHH:MM:SS.000Z",
  "nextBillAmount": 0,
  "cardType": "XXXX",
  "cardDigits": "XXXX",
  "strapSerial": "XXXXXXXXXXXX",
  "checkoutOrigin": "domestic",
  "balance": 0,
  "canUpgrade": false,
  "retentionPromo": false,
  "upgradeStatus": "unneeded",
  "upcycleStatus": "ineligible",
  "coupon": null,
  "promotion": null,
  "affiliate": null,
  "scheduledCancelDate": null,
  "commitmentStart": null,
  "commitmentEnd": null,
  "cancellationOffer": {
    "eligible": false,
    "couponOneMonth50": "CANCEL50ONEMONTH",
    "coupon50": "CANCEL50",
    "coupon100": "FREEMONTH01302020"
  },
  "firstName": "Your",
  "email": "user@email.com"
}
```

---

### Metrics Response

```json
{
  "name": "heart_rate",
  "start": 1771056000000,
  "values": [
    { "time": 1771056000000, "data": 54 },
    { "time": 1771056060010, "data": 53 },
    { "time": 1771056120010, "data": 57 }
  ]
}
```

- `time` is a **Unix millisecond epoch** — convert with `datetime.fromtimestamp(ms / 1000)`
- `data` is the metric value (integer bpm for heart_rate)
- `start` is the query window start as Unix ms epoch
- At `step=60`: one datapoint per minute (~1440 points for a full 24h day)
- At `step=6`: one datapoint every ~6 seconds (~600 points per hour)

---

### Sleep Events Response

Returns a flat array of typed interval objects — one per sleep stage segment:

```json
[
  { "during": "['2026-02-14T06:51:01.940Z','2026-02-14T06:53:42.960Z')", "type": "LATENCY" },
  { "during": "['2026-02-14T06:53:42.960Z','2026-02-14T07:06:45.940Z')", "type": "LIGHT" },
  { "during": "['2026-02-14T07:06:45.940Z','2026-02-14T07:41:18.960Z')", "type": "SWS" },
  { "during": "['2026-02-14T07:53:19.950Z','2026-02-14T08:10:51.010Z')", "type": "REM" },
  { "during": "['2026-02-14T08:10:51.010Z','2026-02-14T08:17:21.010Z')", "type": "WAKE" },
  { "during": "['2026-02-14T11:46:31.450Z','2026-02-14T11:48:01.450Z')", "type": "DISTURBANCES" }
]
```

**Event types:**

| Type | Description |
|---|---|
| `LATENCY` | Time from lights-out to first sleep onset |
| `LIGHT` | Light sleep (N1/N2) |
| `SWS` | Slow-wave sleep — deep / restorative |
| `REM` | REM sleep — dreaming, memory consolidation |
| `WAKE` | Full arousal (you woke up) |
| `DISTURBANCES` | Brief micro-arousal (< 30s) — didn't fully wake |

All `during` fields use PostgreSQL range notation `['start','end')`. Parse with:
```python
import re
parts = re.findall(r"[\d\-T:.Z]+", event["during"])
start, end = parts[0], parts[1]
```

**To get the HR trace alongside stages**, combine this endpoint with:
```
GET /metrics-service/v1/metrics/user/:id
  ?name=heart_rate&start=<sleep_start>&end=<sleep_end>&step=60&order=t&apiVersion=7
```
Then join on timestamp to know which stage each HR reading falls in.

---

## cURL Examples

**Set variables first:**
```bash
TOKEN="your_bearer_token_here"
USER_ID="YOUR_USER_ID"
```

---

**Login and capture token:**
```bash
curl -s -X POST "https://api.prod.whoop.com/api-server/oauth/token" \
  -H "Content-Type: application/json" \
  -d '{
    "grant_type": "password",
    "username": "your@email.com",
    "password": "yourpassword",
    "issueRefresh": true
  }' | python3 -m json.tool
```

---

**Fetch last 2 weeks of cycles/strain (PST user, "2w" ending 2026-02-16):**
```bash
curl -s "https://api.prod.whoop.com/core-details-bff/v0/cycles/details?apiVersion=7&id=$USER_ID&startTime=2026-02-03T08:00:00.000Z&endTime=2026-02-16T07:59:59.999Z&limit=12" \
  -H "Authorization: bearer $TOKEN" | python3 -m json.tool
```

---

**Fetch current day's cycle:**
```bash
curl -s "https://api.prod.whoop.com/core-details-bff/v0/cycles/details?apiVersion=7&id=$USER_ID&startTime=2026-02-24T08:00:00.000Z&endTime=2026-02-25T07:59:59.999Z" \
  -H "Authorization: bearer $TOKEN" | python3 -m json.tool
```

---

**Fetch user profile:**
```bash
curl -s "https://api.prod.whoop.com/users-service/v2/bootstrap/?apiVersion=7&accountType=users&id=$USER_ID&include=profile&include=teams" \
  -H "Authorization: bearer $TOKEN" | python3 -m json.tool
```

---

**Fetch sports catalog (map sport_id → name):**
```bash
curl -s "https://api.prod.whoop.com/activities-service/v1/sports/history?apiVersion=7" \
  -H "Authorization: bearer $TOKEN" | python3 -m json.tool
```

---

**Fetch membership info:**
```bash
curl -s "https://api.prod.whoop.com/membership" \
  -H "Authorization: bearer $TOKEN" | python3 -m json.tool
```

---

**Refresh token:**
```bash
curl -s -X POST "https://api.prod.whoop.com/api-server/oauth/token" \
  -H "Content-Type: application/json" \
  -d "{\"grant_type\": \"refresh_token\", \"refresh_token\": \"$REFRESH_TOKEN\", \"issueRefresh\": false}" \
  | python3 -m json.tool
```

---

## Notes & Gotchas

1. **`apiVersion=7` is required** — every request needs this query param. The JS source confirms all requests are automatically injected with it.

2. **`id=<userId>` on cycles endpoint** — the user ID goes as a query param (`&id=YOUR_USER_ID`), not a path parameter.

3. **lowercase `bearer`** — the auth header must be `authorization: bearer <token>`, not `Bearer`.

4. **Timezone-aware dates** — day boundaries are computed from the user's local timezone. Passing midnight UTC will return wrong data if the user is not in UTC. Get the user's `timezone_offset` from the bootstrap endpoint first.

5. **Range string format** — `during`, `days`, `optimal_sleep_times` use PostgreSQL range notation: `['start','end')` (inclusive start, exclusive end). Parse with a regex to extract the timestamps.

6. **Sleep durations are in milliseconds** — divide by 1000 for seconds, 60000 for minutes, 3600000 for hours.

7. **Workout zone durations are in seconds** — `zone_durations_v2.*_duration` values.

8. **`hrv_rmssd` is in seconds** — multiply by 1000 to get the conventional millisecond (ms) display value.

9. **`sport_id` mapping** — the `workout.sport_id` integer maps to the sports catalog. Fetch `/activities-service/v1/sports/history` once and build a lookup table.

10. **HAR files strip auth headers** — Chrome removes `Authorization` and `Cookie` from saved HAR files. The header is confirmed present in real requests via JS source analysis. Always send the token.

11. **These are internal endpoints** — WHOOP may change them without notice. For production use, consider the official public API at `developer.whoop.com`.

12. **`account_id` vs `user_id`** — bootstrap returns two IDs. The `user_id` (`YOUR_USER_ID`) is what you use in API calls. The `account.id` (`YOUR_ACCOUNT_ID`) is the internal account identifier.
