# Ospex API v1 Reference

## Base URL

```
https://api.ospex.org/v1
```

All read endpoints use `GET`. Instant match endpoints use `POST`. No authentication is required.

## Rate Limiting

100 requests per 60 seconds per IP. Exceeding the limit returns:

```json
{ "error": "Too many requests, please try again later.", "code": "RATE_LIMIT_EXCEEDED" }
```

Rate limit headers follow the `draft-7` standard (`RateLimit-Limit`, `RateLimit-Remaining`, `RateLimit-Reset`).

## Response Format

Every successful response uses a dual-format envelope:

```json
{
  "data": { ... },
  "formatted": "Human-readable summary text"
}
```

Add `?format=raw` to any request to omit the `formatted` field and return `{ "data": ... }` only.

## Error Format

```json
{
  "error": "Description of what went wrong.",
  "code": "ERROR_CODE"
}
```

Common codes:

| Code | HTTP Status | Meaning |
|------|-------------|---------|
| `INVALID_PARAM` | 400 | Malformed or out-of-range parameter |
| `NOT_FOUND` | 404 | Resource does not exist |
| `RATE_LIMIT_EXCEEDED` | 429 | Too many requests |
| `INTERNAL_ERROR` | 500 | Unhandled server error |

## Team Name Slugs

Endpoints that accept team names use URL-friendly slugs:
- Lowercase, hyphens instead of spaces
- Apostrophes and periods stripped
- Example: `Saint Mary's` -> `saint-marys`
- Example: `Texas A&M` -> `texas-am`
- Team names often include mascots: `Duke Blue Devils` -> `duke-blue-devils`

---

## Endpoints

### GET /markets

List upcoming on-chain markets.

**Query Parameters:**

| Name | Type | Default | Constraints |
|------|------|---------|-------------|
| `sport` | string | _(all)_ | `nba`, `nhl`, `ncaab`, `nfl`, `mlb` (case-insensitive) |
| `status` | string | _(all)_ | Filter by contest status (case-insensitive) |
| `window` | number | 72 | Hours lookahead, range 1-168 |

**curl:**

```bash
curl "https://api.ospex.org/v1/markets?sport=nba&window=48"
```

**Response shape:**

```json
{
  "data": [
    {
      "contestId": "42",
      "awayTeam": "Celtics",
      "homeTeam": "Knicks",
      "sport": "nba",
      "sportId": 1,
      "matchTime": "2026-02-22T00:00:00.000Z",
      "status": "Upcoming",
      "speculations": [
        {
          "speculationId": "101",
          "type": "moneyline",
          "theNumber": null,
          "speculationStatus": 0
        },
        {
          "speculationId": "102",
          "type": "spread",
          "theNumber": -3.5,
          "speculationStatus": 0
        }
      ]
    }
  ],
  "formatted": "..."
}
```

**Errors:**
- 400 if `sport` is not one of the supported values
- 400 if `window` is outside 1-168
- Returns empty array (not 404) if no markets match

---

### GET /markets/:contestId

Single market with orderbook depth.

**Path Parameters:**

| Name | Type | Required |
|------|------|----------|
| `contestId` | string | yes |

**curl:**

```bash
curl "https://api.ospex.org/v1/markets/42"
```

**Response shape:**

```json
{
  "data": {
    "contestId": "42",
    "awayTeam": "Celtics",
    "homeTeam": "Knicks",
    "sport": "nba",
    "sportId": 1,
    "matchTime": "2026-02-22T00:00:00.000Z",
    "status": "Upcoming",
    "speculations": [
      {
        "speculationId": "101",
        "type": "moneyline",
        "theNumber": null,
        "speculationStatus": 0,
        "orderbook": [
          {
            "positionId": "501",
            "side": "upper",
            "makerOdds": 1.85,
            "takerOdds": 2.17,
            "amountUSDC": 3.0
          }
        ]
      }
    ]
  },
  "formatted": "..."
}
```

**Fields:**
- `orderbook` lists all unmatched positions for each speculation
- `side`: `"upper"` = away/over, `"lower"` = home/under
- `makerOdds`: decimal odds offered by the position maker
- `takerOdds`: decimal odds available to whoever takes the other side
- `amountUSDC`: unmatched liquidity in USDC

**Errors:**
- 400 if `contestId` is missing
- 404 if contest does not exist

---

### GET /odds

Best available on-chain prices per side (summary view, not full orderbook).

**Query Parameters:**

| Name | Type | Default | Constraints |
|------|------|---------|-------------|
| `sport` | string | _(all)_ | `nba`, `nhl`, `ncaab`, `nfl`, `mlb` |
| `window` | number | 24 | Hours lookahead, range 1-168 |

**curl:**

```bash
curl "https://api.ospex.org/v1/odds?sport=nba"
```

**Response shape:**

```json
{
  "data": [
    {
      "contestId": "42",
      "awayTeam": "Celtics",
      "homeTeam": "Knicks",
      "sport": "nba",
      "matchTime": "2026-02-22T00:00:00.000Z",
      "speculations": [
        {
          "type": "moneyline",
          "theNumber": null,
          "upper": {
            "bestOdds": 2.15,
            "availableUSDC": 6.0
          },
          "lower": {
            "bestOdds": 1.85,
            "availableUSDC": 9.0
          }
        }
      ]
    }
  ],
  "formatted": "..."
}
```

**Fields:**
- `upper` = away team or over side
- `lower` = home team or under side
- `bestOdds`: best decimal odds available to a taker on that side, or `null` if no liquidity
- `availableUSDC`: total unmatched USDC on that side

**Errors:**
- 400 if `sport` or `window` is invalid
- Returns empty array if no odds available

---

### GET /analytics/elo/:teamName

ELO rating for a single team.

**Path Parameters:**

| Name | Type | Required |
|------|------|----------|
| `teamName` | string (slug) | yes |

**Query Parameters:**

| Name | Type | Default | Constraints |
|------|------|---------|-------------|
| `sport` | string | `ncaab` | `nba`, `nhl`, `ncaab` |
| `season` | number | _(current)_ | Range 2000-2100 |

**curl:**

```bash
curl "https://api.ospex.org/v1/analytics/elo/duke?sport=ncaab"
```

**Response shape:**

```json
{
  "data": {
    "teamName": "Duke Blue Devils",
    "teamId": "3a50945b-98dd-44b8-a75d-ee524bbe2106",
    "eloRating": 1972,
    "nationalRank": 2,
    "trend": 122,
    "trendLabel": "rising",
    "gamesPlayed": 25,
    "wins": 23,
    "losses": 2,
    "winsVsTop50": 9,
    "lossesVsTop50": 2,
    "lastCalculated": "2026-02-20T13:00:01.258+00:00",
    "slug": "duke-blue-devils"
  },
  "formatted": "..."
}
```

**Fields:**
- `nationalRank`: AP Poll or Selection Sunday ranking, or `null` if unranked
- `trend`: ELO rating change trend (positive = improving)
- `trendLabel`: Human-readable trend direction (`"rising"`, `"falling"`, `"stable"`)
- `winsVsTop50` / `lossesVsTop50`: Record against top-50 ELO teams
- `slug`: URL-safe version of the team name (use this in requests)

**Errors:**
- 400 if `sport` is not supported or `season` is out of range
- 404 if team not found

---

### GET /analytics/elo/rankings

Top ELO rankings for a sport.

**Query Parameters:**

| Name | Type | Default | Constraints |
|------|------|---------|-------------|
| `sport` | string | `ncaab` | `nba`, `nhl`, `ncaab` |
| `season` | number | _(current)_ | Range 2000-2100 |
| `top` | number | _(all)_ | Range 1-500 |

**curl:**

```bash
curl "https://api.ospex.org/v1/analytics/elo/rankings?sport=ncaab&top=25"
```

**Response shape:**

```json
{
  "data": [
    {
      "teamName": "Michigan Wolverines",
      "teamId": "6f8055a6-a930-4a6e-a3c9-e7ff70636535",
      "eloRating": 2004,
      "nationalRank": 1,
      "trend": 178,
      "trendLabel": "rising",
      "gamesPlayed": 25,
      "wins": 24,
      "losses": 1,
      "winsVsTop50": 6,
      "lossesVsTop50": 1,
      "lastCalculated": "2026-02-20T13:00:01.258+00:00",
      "slug": "michigan-wolverines"
    }
  ],
  "formatted": "..."
}
```

**Errors:**
- 400 if `sport`, `season`, or `top` is invalid
- 404 if no rankings found for the given sport/season

---

### GET /analytics/elo/all

All ELO ratings for a sport (no top limit).

**Query Parameters:**

| Name | Type | Default | Constraints |
|------|------|---------|-------------|
| `sport` | string | `ncaab` | `nba`, `nhl`, `ncaab` |
| `season` | number | _(current)_ | Range 2000-2100 |

**curl:**

```bash
curl "https://api.ospex.org/v1/analytics/elo/all?sport=nba"
```

**Response shape:** Same as `/analytics/elo/rankings` (array of team ELO objects with `teamId`, `trend`, `trendLabel`, `gamesPlayed`, `winsVsTop50`, `lossesVsTop50`).

**Errors:**
- 400 if `sport` or `season` is invalid
- 404 if no ELO data found

---

### GET /analytics/matchup/:contestId

Comprehensive matchup analysis for a contest. Primary: ELO-based win probability. Supplementary sections (schedule, standings, injuries, rankings, expert picks) are fetched in parallel and included when available.

**Path Parameters:**

| Name | Type | Required |
|------|------|----------|
| `contestId` | string | yes |

**Query Parameters:**

| Name | Type | Default | Constraints |
|------|------|---------|-------------|
| `sport` | string | `ncaab` | `nba`, `nhl`, `ncaab` |

**curl:**

```bash
curl "https://api.ospex.org/v1/analytics/matchup/87?sport=ncaab"
```

**Response shape:**

```json
{
  "data": {
    "contestId": "87",
    "away": {
      "teamName": "Kentucky Wildcats",
      "slug": "kentucky-wildcats",
      "eloRating": 1731,
      "nationalRank": 46,
      "record": "17-8"
    },
    "home": {
      "teamName": "Auburn Tigers",
      "slug": "auburn-tigers",
      "eloRating": 1633,
      "nationalRank": 79,
      "record": "14-11"
    },
    "prediction": {
      "awayWinProbability": 0.497,
      "homeWinProbability": 0.503,
      "favoredTeam": "Auburn Tigers",
      "effectiveEloDiff": 2
    },
    "schedule": {
      "away": { "recentRecord": "0-1 last 1", "daysRest": 7, "isBackToBack": false, "lastGameDate": "...", "lastGameResult": "L 83-92 vs Florida Gators" },
      "home": { "recentRecord": "0-1 last 1", "daysRest": 7, "isBackToBack": false, "lastGameDate": "...", "lastGameResult": "L 75-88 vs Arkansas Razorbacks" }
    },
    "standings": {
      "away": { "record": "17-8", "conferenceRank": 5, "streak": "L1", "homeRecord": "12-2", "awayRecord": "5-6" },
      "home": { "record": "14-11", "conferenceRank": 8, "streak": "L2", "homeRecord": "10-3", "awayRecord": "4-8" }
    },
    "injuries": {
      "away": [{ "player": "...", "position": "G", "status": "Out" }],
      "home": []
    },
    "rankings": {
      "away": { "apRank": 12, "coachesRank": 14 },
      "home": { "apRank": null, "coachesRank": null }
    },
    "expertPicks": [
      { "pundit": "John Smith", "org": "CBS", "pick": "Kentucky", "pickType": "winner", "reasoning": "..." }
    ]
  },
  "formatted": "..."
}
```

**Fields:**
- `prediction.effectiveEloDiff`: ELO difference including +100 home-court advantage
- Win probabilities sum to 1.0 and are rounded to 3 decimal places
- `schedule`: rest days, back-to-back status, recent form. Included for all sports.
- `standings`: record, conference rank, streaks. NBA/NHL only (omitted for NCAAB).
- `injuries`: current injury report. NBA/NHL only (omitted for NCAAB).
- `rankings`: AP/Coaches poll rank. NCAAB only (omitted for NBA/NHL).
- `expertPicks`: pundit picks if available for this contest.
- Supplementary sections are silently omitted if data is unavailable — never errors.

**Errors:**
- 400 if `contestId` is missing or `sport` is invalid
- 404 if contest not found or ELO data missing for either team

---

### GET /analytics/schedule/:teamName

Recent and upcoming games, rest days, and back-to-back status.

**Path Parameters:**

| Name | Type | Required |
|------|------|----------|
| `teamName` | string (slug) | yes |

**Query Parameters:**

| Name | Type | Required | Constraints |
|------|------|----------|-------------|
| `sport` | string | **yes** | `nba`, `nhl`, `ncaab` |

**curl:**

```bash
curl "https://api.ospex.org/v1/analytics/schedule/duke-blue-devils?sport=ncaab"
```

**Response shape:**

```json
{
  "data": {
    "teamName": "Duke Blue Devils",
    "sport": "ncaab",
    "recentGames": [
      { "gameId": "401820740", "date": "2026-02-14T17:00:00+00:00", "opponent": "Clemson Tigers", "location": "home", "status": "final", "score": "67-54", "result": "W" }
    ],
    "upcomingGames": [
      { "gameId": "401820751", "date": "2026-02-22T23:30:00+00:00", "opponent": "Syracuse Orange", "location": "away", "status": "scheduled", "score": null, "result": null }
    ],
    "daysRest": 7,
    "isBackToBack": false,
    "lastGameDate": "2026-02-14T17:00:00+00:00"
  },
  "formatted": "..."
}
```

**Errors:**
- 400 if `sport` is missing or invalid
- 404 if team not found

---

### GET /analytics/injuries/:teamName

Current injury report. Empty array means team is healthy.

**Path Parameters:**

| Name | Type | Required |
|------|------|----------|
| `teamName` | string (slug) | yes |

**Query Parameters:**

| Name | Type | Required | Constraints |
|------|------|----------|-------------|
| `sport` | string | **yes** | `nba`, `nhl` |

**curl:**

```bash
curl "https://api.ospex.org/v1/analytics/injuries/los-angeles-lakers?sport=nba"
```

**Response shape:**

```json
{
  "data": {
    "teamName": "Los Angeles Lakers",
    "sport": "nba",
    "injuries": [
      { "player": "LeBron James", "position": "F", "status": "Day-To-Day", "injuryType": "Knee" }
    ],
    "lastUpdated": "2026-02-21T01:45:12.787+00:00"
  },
  "formatted": "..."
}
```

**Errors:**
- 400 if `sport` is missing or not `nba`/`nhl`
- 404 if team not found

---

### GET /analytics/standings

Full league standings sorted by conference rank.

**Query Parameters:**

| Name | Type | Required | Constraints |
|------|------|----------|-------------|
| `league` | string | **yes** | `nba`, `nhl` |
| `conference` | string | no | Filter by conference name (e.g., `Eastern`, `Western`) |

**curl:**

```bash
curl "https://api.ospex.org/v1/analytics/standings?league=nba"
curl "https://api.ospex.org/v1/analytics/standings?league=nhl&conference=Western"
```

**Response shape:**

```json
{
  "data": {
    "league": "nba",
    "conference": null,
    "standings": [
      {
        "teamName": "Detroit Pistons",
        "teamAbbrev": "DET",
        "conference": "Eastern",
        "wins": 41, "losses": 13,
        "winPct": 0.759,
        "conferenceRank": 1,
        "gamesBack": 0,
        "streak": "W4",
        "homeRecord": "21-6",
        "awayRecord": "19-7",
        "last10": "8-2",
        "pointDifferential": 437
      }
    ]
  },
  "formatted": "..."
}
```

**NHL-specific fields:** `otl` (overtime losses), `points` (standings points).

**Errors:**
- 400 if `league` is missing, invalid, or `ncaab` (use `/analytics/rankings` for NCAAB)

---

### GET /analytics/team-stats/:teamName

Team statistics — point differential, PPG, shooting percentages, special teams.

**Path Parameters:**

| Name | Type | Required |
|------|------|----------|
| `teamName` | string (slug) | yes |

**Query Parameters:**

| Name | Type | Required | Constraints |
|------|------|----------|-------------|
| `sport` | string | **yes** | `nba`, `nhl` |

**curl:**

```bash
curl "https://api.ospex.org/v1/analytics/team-stats/los-angeles-lakers?sport=nba"
```

**Response shape:**

```json
{
  "data": {
    "teamName": "Los Angeles Lakers",
    "sport": "nba",
    "gamesPlayed": 55,
    "pointDifferential": 0.1,
    "pointsPerGame": 116.18,
    "pointsAllowedPerGame": 116.1,
    "pace": null,
    "fieldGoalPct": 50.0,
    "threePointPct": 35.3,
    "powerPlayPct": null,
    "penaltyKillPct": null,
    "savePct": null,
    "lastUpdated": "2026-02-21T12:52:10.527+00:00"
  },
  "formatted": "..."
}
```

**NBA fields:** `fieldGoalPct`, `threePointPct` (as percentages, e.g., 50.0 = 50.0%).
**NHL fields:** `pace` (shots/game), `powerPlayPct`, `penaltyKillPct`, `savePct`.

**Errors:**
- 400 if `sport` is missing or not `nba`/`nhl`
- 404 if team not found

---

### GET /analytics/rankings

NCAAB AP Top 25 or Coaches Poll rankings.

**Query Parameters:**

| Name | Type | Default | Constraints |
|------|------|---------|-------------|
| `poll` | string | `ap` | `ap`, `coaches` |

**curl:**

```bash
curl "https://api.ospex.org/v1/analytics/rankings?poll=ap"
```

**Response shape:**

```json
{
  "data": {
    "pollName": "AP Top 25",
    "weekNumber": 14,
    "rankings": [
      {
        "rank": 1,
        "teamName": "Arizona Wildcats",
        "teamAbbrev": "ARIZ",
        "previousRank": 1,
        "rankChange": 0,
        "points": 1475,
        "firstPlaceVotes": 59
      }
    ],
    "lastUpdated": "2026-02-17T17:30:00.000+00:00"
  },
  "formatted": "..."
}
```

**Errors:**
- 400 if `poll` is not `ap` or `coaches`

---

### GET /analytics/rosters/:teamName

Current team roster with positions, jersey numbers, height, and age.

**Path Parameters:**

| Name | Type | Required |
|------|------|----------|
| `teamName` | string (slug) | yes |

**Query Parameters:**

| Name | Type | Required | Constraints |
|------|------|----------|-------------|
| `sport` | string | **yes** | `nba`, `nhl` |

**curl:**

```bash
curl "https://api.ospex.org/v1/analytics/rosters/los-angeles-lakers?sport=nba"
```

**Response shape:**

```json
{
  "data": {
    "teamName": "Los Angeles Lakers",
    "sport": "nba",
    "roster": [
      { "player": "LeBron James", "position": "F", "jersey": "23", "height": "6' 9\"", "age": 41 }
    ],
    "lastUpdated": "2026-02-20T12:00:00.000+00:00"
  },
  "formatted": "..."
}
```

**Errors:**
- 400 if `sport` is missing or not `nba`/`nhl`
- 404 if team not found

---

### GET /analytics/expert-picks/:contestId

Attributed expert/pundit picks for a contest. Returns empty array if no picks available.

**Path Parameters:**

| Name | Type | Required |
|------|------|----------|
| `contestId` | string | yes |

**curl:**

```bash
curl "https://api.ospex.org/v1/analytics/expert-picks/87"
```

**Response shape:**

```json
{
  "data": {
    "contestId": "87",
    "picks": [
      {
        "pundit": "John Smith",
        "org": "CBS Sports",
        "pick": "Kentucky",
        "pickType": "winner",
        "confidence": "high",
        "reasoning": "Kentucky's defense has been dominant in SEC play",
        "sourceUrl": "https://www.cbssports.com/..."
      }
    ]
  },
  "formatted": "..."
}
```

**Errors:**
- 400 if `contestId` is missing
- Returns empty `picks` array (not 404) if no picks available

---

### GET /analytics/odds-history/:contestId

Opening lines, current lines, and line movement for a contest.

**Path Parameters:**

| Name | Type | Required |
|------|------|----------|
| `contestId` | string | yes |

**curl:**

```bash
curl "https://api.ospex.org/v1/analytics/odds-history/87"
```

**Response shape:**

```json
{
  "data": {
    "contestId": "87",
    "opener": {
      "spread": { "line": -2.5, "awayOdds": 1.909, "homeOdds": 1.909 },
      "total": { "line": 157.5, "overOdds": 1.909, "underOdds": 1.909 },
      "moneyline": { "awayOdds": 2.28, "homeOdds": 1.654 },
      "capturedAt": "2026-02-21T05:00:05.013+00:00"
    },
    "current": {
      "spread": { "line": -3.5, "awayOdds": 1.909, "homeOdds": 1.909 },
      "total": { "line": 157, "overOdds": 1.926, "underOdds": 1.901 },
      "moneyline": { "awayOdds": 2.47, "homeOdds": 1.588 },
      "capturedAt": "2026-02-21T19:00:05.677+00:00"
    },
    "lineMovement": {
      "spread": { "opened": -2.5, "current": -3.5, "moved": -1 },
      "total": { "opened": 157.5, "current": 157, "moved": -0.5 }
    }
  },
  "formatted": "..."
}
```

**Fields:**
- `opener`: first captured odds snapshot
- `current`: most recent odds snapshot
- `lineMovement`: how spread and total lines have moved since opening
- All odds are in decimal format

**Errors:**
- 400 if `contestId` is missing
- 404 if contest not found or no odds data available

---

### GET /protocol/info

Static protocol metadata — network, contracts, supported sports, fees.

**curl:**

```bash
curl "https://api.ospex.org/v1/protocol/info"
```

**Response shape:**

```json
{
  "data": {
    "name": "Ospex",
    "network": "polygon",
    "chainId": 137,
    "contracts": {
      "core": "0x8016b2C5f161e84940E25Bb99479aAca19D982aD",
      "usdc": "0x3c499c542cEF5E3811e1192ce70d8cC03d5c3359"
    },
    "supportedSports": ["NBA", "NHL", "NCAAB"],
    "fees": {
      "platformFeePct": 0,
      "description": "No platform fees. 100% of stakes go to winners."
    },
    "leaderboard": {
      "active": true,
      "description": "Weekly competitions with USDC prize pools."
    }
  },
  "formatted": "..."
}
```

---

### GET /protocol/agents

Known automated agents with descriptions and last run times.

**curl:**

```bash
curl "https://api.ospex.org/v1/protocol/agents"
```

**Response shape:**

```json
{
  "data": {
    "agents": [
      {
        "agentId": "market_maker_michelle",
        "network": "polygon",
        "lastRun": "2026-02-21T14:15:43.430Z",
        "description": "Market maker agent. Provides liquidity by posting positions on both sides."
      },
      {
        "agentId": "degen_dan",
        "network": "polygon",
        "lastRun": "2026-02-21T19:14:59.613Z",
        "description": "Aggressive bettor agent. Takes positions on favorites."
      }
    ]
  },
  "formatted": "..."
}
```

---

### GET /leaderboard

Current active leaderboard standings.

**Query Parameters:**

| Name | Type | Default | Constraints |
|------|------|---------|-------------|
| `top` | number | 50 | Range 1-200 |

**curl:**

```bash
curl "https://api.ospex.org/v1/leaderboard?top=10"
```

**Response shape:**

```json
{
  "data": {
    "leaderboardId": "7",
    "name": "February 2026",
    "startTime": "2026-02-01T00:00:00.000Z",
    "endTime": "2026-02-28T23:59:59.000Z",
    "entries": [
      {
        "rank": 1,
        "address": "0xabcd...1234",
        "declaredBankroll": 250.0
      }
    ]
  },
  "formatted": "..."
}
```

**Fields:**
- `entries` sorted by `declaredBankroll` descending
- `rank` is 1-based
- `declaredBankroll` is in USDC

**Special case:** Returns `{ "data": null, "formatted": "No active leaderboard found." }` if no leaderboard is currently active.

**Errors:**
- 400 if `top` is outside 1-200

---

### GET /leaderboard/:address

Position history for a wallet address.

**Path Parameters:**

| Name | Type | Required |
|------|------|----------|
| `address` | string | yes |

**Query Parameters:**

| Name | Type | Default | Constraints |
|------|------|---------|-------------|
| `limit` | number | 50 | Range 1-200 |
| `offset` | number | 0 | Must be >= 0 |

**curl:**

```bash
curl "https://api.ospex.org/v1/leaderboard/0xabcd1234abcd1234abcd1234abcd1234abcd1234?limit=10"
```

**Response shape:**

```json
{
  "data": {
    "address": "0xabcd1234abcd1234abcd1234abcd1234abcd1234",
    "totalPositions": 47,
    "matchedPositions": 38,
    "totalMatchedUSDC": 114.0,
    "totalUnmatchedUSDC": 27.0,
    "positions": [
      {
        "speculationId": "101",
        "positionType": 0,
        "matchedAmountUSDC": 3.0,
        "unmatchedAmountUSDC": 0.0,
        "claimed": true,
        "upperOdds": 1.85,
        "lowerOdds": 2.17
      }
    ],
    "pagination": {
      "limit": 10,
      "offset": 0,
      "hasMore": true
    }
  },
  "formatted": "..."
}
```

**Fields:**
- `positionType`: `0` = upper (away/over), `1` = lower (home/under)
- `claimed`: whether winnings have been withdrawn
- `upperOdds`/`lowerOdds`: decimal odds for each side of the position
- `pagination.hasMore`: `true` if more positions exist beyond the current page

**Errors:**
- 400 if address is not a valid 42-character 0x-prefixed Ethereum address
- 400 if `limit` or `offset` is out of range
- Returns empty `positions` array (not 404) if address has no history

---

### POST /instant-match/:speculationId/quote

Request an instant match quote via Server-Sent Events (SSE). The response streams progress updates and concludes with an approved, rejected, or error result.

**Path Parameters:**

| Name | Type | Required |
|------|------|----------|
| `speculationId` | string | yes |

**Request body:**

```json
{
  "feeTxHash": "0x...",
  "side": "away",
  "amountUSDC": 3.0,
  "odds": 1.85,
  "oddsFormat": "decimal",
  "wallet": "0xYourWalletAddress"
}
```

**Fields:**
- `feeTxHash`: Transaction hash of a 0.20 USDC fee payment sent to the fee wallet (`0xdaC630aE52b868FF0A180458eFb9ac88e7425114`) before requesting the quote
- `side`: One of `over`, `under`, `home`, `away`
- `amountUSDC`: Requested position size in USDC
- `odds`: Requested odds in the format specified by `oddsFormat`
- `oddsFormat`: `decimal` (default), `american`, or `probability`
- `wallet`: Your wallet address (0x-prefixed)

**SSE event types:**

| Event | Description |
|-------|-------------|
| `progress` | Intermediate update with `step`, `message`, and optional `details` |
| `result` | Final outcome — either approved or rejected (see shapes below) |
| `error` | Evaluation failed — includes `error` message and `code` |
| `: keepalive` | Comment line sent every 15 seconds to keep the connection open |

**Approved result shape:**

```json
{
  "approved": true,
  "quoteId": "uuid",
  "approvedOddsDecimal": 1.85,
  "approvedOddsAmerican": -118,
  "expiresAt": "2026-02-22T00:01:00.000Z"
}
```

An approved result may include a `counterOffer` if the evaluator suggests different odds:

```json
{
  "approved": true,
  "quoteId": "uuid",
  "approvedOddsDecimal": 1.90,
  "approvedOddsAmerican": -111,
  "counterOffer": {
    "oddsDecimal": 1.90,
    "oddsAmerican": -111,
    "ttlSeconds": 60
  },
  "expiresAt": "2026-02-22T00:01:00.000Z"
}
```

**Rejected result shape:**

```json
{
  "approved": false,
  "reason": "Odds too far from market"
}
```

**Pre-SSE errors (JSON, before streaming begins):**

| Code | HTTP | Meaning |
|------|------|---------|
| `INVALID_REQUEST` | 400 | Missing or invalid field in request body |
| `INVALID_ODDS` | 400 | Odds conversion failed or result <= 1.0 |
| `SPECULATION_NOT_FOUND` | 404 | Speculation does not exist |
| `INVALID_SPECULATION` | 400 | Unknown market type for the speculation |

**SSE errors (streamed after connection opens):**

| Code | Meaning |
|------|---------|
| `INVALID_FEE_TX` | Fee transaction verification failed |
| `FEE_TX_REPLAY` | Fee transaction hash already used |
| `NOT_AVAILABLE` | Market maker is not available for this market |
| `BUSY` | Market maker is evaluating another request |
| `QUOTE_CREATION_FAILED` | Internal error creating the quote |
| `EVALUATION_ERROR` | LLM evaluation failed |

---

### POST /instant-match/:quoteId/match

Execute a match against an approved quote. Call this after creating your position on-chain.

**Path Parameters:**

| Name | Type | Required |
|------|------|----------|
| `quoteId` | string | yes |

**Request body:**

```json
{
  "positionId": "123"
}
```

**Fields:**
- `positionId`: The on-chain position ID created by the user

**Success response (200):**

```json
{
  "matched": true,
  "txHash": "0x..."
}
```

**Errors:**

| Code | HTTP | Meaning |
|------|------|---------|
| `INVALID_REQUEST` | 400 | Missing or invalid `positionId` |
| `QUOTE_NOT_FOUND` | 404 | Quote does not exist |
| `QUOTE_EXPIRED` | 410 | Quote TTL has elapsed |
| `INVALID_QUOTE_STATE` | 400 | Quote is not in a matchable state |
| `POSITION_NOT_FOUND` | 400 | Position does not exist on-chain |
| `POSITION_MISMATCH` | 400 | Position does not match the quote parameters |
| `STAKE_TOO_SMALL` | 400 | Position stake is below the minimum |
| `MATCH_EXECUTION_FAILED` | 500 | On-chain match transaction failed |
