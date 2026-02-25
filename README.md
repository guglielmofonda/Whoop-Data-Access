# WHOOP Personal Data Reference

A reference for accessing your own personal health data from WHOOP programmatically. Documents the web endpoints that the WHOOP app uses to display your data, observed through your own authenticated browser session.

> **Note**: WHOOP offers an [official developer API](https://developer.whoop.com/) and a personal data export feature — those are the recommended paths for most use cases. This guide is a personal reference for accessing your own health data. It is not intended for accessing other people's data, bulk collection, or commercial use. Always refer to the [WHOOP Terms of Use](https://www.whoop.com/us/en/whoop-terms-of-use/) before building anything beyond personal use.

---

## What's in here

| File | Description |
|---|---|
| [`WHOOP_API_GUIDE.md`](./WHOOP_API_GUIDE.md) | Full endpoint reference — request params, response schemas, code examples |

---

## Quick Start

### 1. Get your access token

**Option A — From the browser (no login request needed):**
1. Log in to [app.whoop.com](https://app.whoop.com)
2. Open DevTools → **Application** → **Local Storage** → `https://app.whoop.com`
3. Copy `security.accessToken`

**Option B — Authenticate directly:**
```bash
curl -s -X POST "https://api.prod.whoop.com/api-server/oauth/token" \
  -H "Content-Type: application/json" \
  -d '{
    "grant_type": "password",
    "username": "your@email.com",
    "password": "yourpassword",
    "issueRefresh": true
  }' | python3 -c "import sys,json; d=json.load(sys.stdin); print(d['access_token'])"
```

### 2. Find your user ID

```bash
TOKEN="your_token_here"

curl -s "https://api.prod.whoop.com/users-service/v2/bootstrap/?apiVersion=7&accountType=users&id=0" \
  -H "Authorization: bearer $TOKEN" | python3 -c "import sys,json; d=json.load(sys.stdin); print(d['user']['id'])"
```

### 3. Fetch your last 2 weeks of data

```bash
USER_ID="your_user_id"

curl -s "https://api.prod.whoop.com/core-details-bff/v0/cycles/details?apiVersion=7&id=$USER_ID&startTime=2026-02-03T08:00:00.000Z&endTime=2026-02-16T07:59:59.999Z&limit=14" \
  -H "Authorization: bearer $TOKEN" | python3 -m json.tool
```

---

## Key Endpoints

| Endpoint | What it returns |
|---|---|
| `POST /api-server/oauth/token` | Login / refresh token |
| `GET /core-details-bff/v0/cycles/details` | Daily cycles: strain, recovery, sleep, workouts |
| `GET /users-service/v2/bootstrap/` | User profile, membership, preferences |
| `GET /sleep-service/v1/sleep-events` | Per-session sleep stage timeline (LIGHT/SWS/REM/WAKE) |
| `GET /metrics-service/v1/metrics/user/:id` | Heart rate & metric time series |
| `GET /activities-service/v1/sports/history` | Full catalog of sport types |
| `GET /membership` | Subscription and billing status |

---

## Technical Notes

- **`apiVersion=7`** must be appended to every request as a query parameter
- **`Authorization: bearer <token>`** — lowercase `bearer`
- **`id=<userId>`** is a query param on cycles/details, not a path param
- Dates are **timezone-aware** — day boundaries are in the user's local timezone, not UTC
- The `metrics-service` returns timestamps as **Unix millisecond epochs** (`{time, data}`), not ISO strings

---

## Response Shapes at a Glance

### Cycles/Details
```json
{
  "records": [{
    "cycle":    { "scaled_strain": 13.8, "day_kilojoules": 6174, ... },
    "sleeps":   [{ "score": 83, "quality_duration": 23399780, ... }],
    "recovery": { "recovery_score": 66, "hrv_rmssd": 0.0778, ... },
    "workouts": [{ "sport_id": 45, "score": 13.1, "kilojoules": 664, ... }],
    "v2_activities": [{ "type": "weightlifting", "score_type": "CARDIO", ... }]
  }]
}
```

### Sleep Events
```json
[
  { "during": "['2026-02-14T06:51:01.940Z','2026-02-14T07:06:45.940Z')", "type": "LIGHT" },
  { "during": "['2026-02-14T07:06:45.940Z','2026-02-14T07:41:18.960Z')", "type": "SWS" },
  { "during": "['2026-02-14T07:53:19.950Z','2026-02-14T08:10:51.010Z')", "type": "REM" }
]
```

### Heart Rate Time Series
```json
{
  "name": "heart_rate",
  "start": 1771056000000,
  "values": [
    { "time": 1771056000000, "data": 54 },
    { "time": 1771056060010, "data": 53 }
  ]
}
```

---

## How This Was Documented

This reference was produced by:
1. Reading the JavaScript served to the browser at `app.whoop.com` to understand endpoint paths, parameter names, and request formatting
2. Capturing the browser's network traffic (HAR export) while logged in as the account owner, then documenting the observed request and response shapes

---

## See Also

- [WHOOP Developer API](https://developer.whoop.com/) — the official public API with OAuth2 PKCE support
- [WHOOP Terms of Use](https://www.whoop.com/us/en/whoop-terms-of-use/)
