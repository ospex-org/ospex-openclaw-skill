# Ospex API v1 Reference

## Base URL

```
https://api.ospex.org/v1
```

All read endpoints use `GET`. Instant match endpoints use `POST`. No authentication required.

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

## Fees

Platform fees are **currently disabled**. The `feeTxHash` field on instant-match quote requests can be omitted. When fees are re-enabled, a 0.20 USDC payment to the fee wallet (`0xdaC630aE52b868FF0A180458eFb9ac88e7425114`) will be required before requesting a quote.

---

## Markets

### GET /markets

List upcoming on-chain markets.

**Query Parameters:**

| Name | Type | Default | Constraints |
|------|------|---------|-------------|
| `sport` | string | _(all)_ | `nba`, `nhl`, `ncaab`, `nfl`, `mlb` (case-insensitive) |
| `status` | string | _(all)_ | Filter by contest status (case-insensitive) |
| `window` | number | 72 | Hours lookahead, range 1-168 |

```bash
curl "https://api.ospex.org/v1/markets?sport=nba&window=48"
```

**Response:**

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
        { "speculationId": "101", "type": "moneyline", "theNumber": null, "speculationStatus": 0 },
        { "speculationId": "102", "type": "spread", "theNumber": -3.5, "speculationStatus": 0 }
      ]
    }
  ]
}
```

**Notes:**
- `speculationStatus`: `0` = open, `1` = settled
- Returns empty array (not 404) if no markets match

**Errors:** 400 if `sport` is unsupported or `window` is outside 1-168

---

### GET /markets/:contestId

Single market with orderbook depth (all unmatched positions).

```bash
curl "https://api.ospex.org/v1/markets/42"
```

**Response:**

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
          { "positionId": "501", "side": "upper", "makerOdds": 1.85, "takerOdds": 2.17, "amountUSDC": 3.0 }
        ]
      }
    ]
  }
}
```

**Orderbook fields:**
- `side`: `"upper"` = away/over, `"lower"` = home/under
- `makerOdds`: decimal odds the position creator set
- `takerOdds`: decimal odds available to whoever takes the other side
- `amountUSDC`: unmatched liquidity

**Errors:** 400 if `contestId` is missing; 404 if contest does not exist

---

### GET /odds

Best available on-chain prices per side (summary view, not full orderbook).

| Name | Type | Default | Constraints |
|------|------|---------|-------------|
| `sport` | string | _(all)_ | `nba`, `nhl`, `ncaab`, `nfl`, `mlb` |
| `window` | number | 24 | Hours lookahead, range 1-168 |

```bash
curl "https://api.ospex.org/v1/odds?sport=nba"
```

**Response:**

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
          "upper": { "bestOdds": 2.15, "availableUSDC": 6.0 },
          "lower": { "bestOdds": 1.85, "availableUSDC": 9.0 }
        }
      ]
    }
  ]
}
```

**Fields:**
- `bestOdds`: best decimal odds available to a taker on that side, or `null` if no liquidity
- `availableUSDC`: total unmatched USDC on that side

---

## Positions

### GET /positions/:address

Position history for a wallet address.

| Name | Type | Default | Constraints |
|------|------|---------|-------------|
| `limit` | number | 50 | Range 1-200 |
| `offset` | number | 0 | Must be >= 0 |

```bash
curl "https://api.ospex.org/v1/positions/0xabcd1234abcd1234abcd1234abcd1234abcd1234?limit=10"
```

**Response:**

```json
{
  "data": {
    "address": "0xabcd...",
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
    "pagination": { "limit": 10, "offset": 0, "hasMore": true }
  }
}
```

**Fields:**
- `positionType`: `0` = upper (away/over), `1` = lower (home/under)
- `claimed`: whether winnings have been withdrawn
- `upperOdds`/`lowerOdds`: decimal odds for each side

**Errors:** 400 if address is not a valid 42-char 0x-prefixed address; returns empty `positions` array (not 404) if no history

---

### GET /positions/:address/status

Categorized view of unclaimed positions: active bets, claimable winnings, and withdrawable unmatched funds. Designed for quick status checks — positions are enriched with team names, market type, and odds.

```bash
curl "https://api.ospex.org/v1/positions/0xabcd1234abcd1234abcd1234abcd1234abcd1234/status"
```

**Response:**

```json
{
  "data": {
    "address": "0xabcd...",
    "active": [
      {
        "positionId": "112_0xabcd..._10284_1",
        "speculationId": "112",
        "oddsPairId": "10284",
        "positionType": 1,
        "team": "Sacramento Kings",
        "opponent": "Phoenix Suns",
        "market": "moneyline",
        "odds": 3.85,
        "matchedAmountUSDC": 2.18,
        "unmatchedAmountUSDC": 0.82
      }
    ],
    "claimable": [
      {
        "positionId": "104_0xabcd..._10119_1",
        "speculationId": "104",
        "oddsPairId": "10119",
        "positionType": 1,
        "team": "New York Knicks",
        "opponent": "San Antonio Spurs",
        "market": "moneyline",
        "odds": 2.2,
        "matchedAmountUSDC": 2.49,
        "unmatchedAmountUSDC": 0.51,
        "result": "won",
        "estimatedPayoutUSDC": 5.99
      }
    ],
    "withdrawable": [
      {
        "positionId": "112_0xabcd..._10284_1",
        "speculationId": "112",
        "oddsPairId": "10284",
        "positionType": 1,
        "team": "Sacramento Kings",
        "opponent": "Phoenix Suns",
        "market": "moneyline",
        "odds": 3.85,
        "matchedAmountUSDC": 2.18,
        "unmatchedAmountUSDC": 0.82
      }
    ],
    "profileUrl": "https://ospex.org/u/0xabcd..."
  }
}
```

**Categories:**
- **active**: Speculation not settled, matched amount > 0 (live bets where the game hasn't been scored)
- **claimable**: Speculation settled, position won (or push/void/forfeit), or has unmatched remainder. Estimated payout >= 0.01 USDC. Use `claimPosition` on-chain to collect.
- **withdrawable**: Speculation not settled, unmatched amount >= 0.01 USDC. Use `adjustUnmatchedPair` on-chain to retrieve.

**Claimable `result` values:** `"won"`, `"lost"`, `"push"`, `"void"`

**Fields on each position:**
- `positionId`: Firebase document ID (also serves as the composite key for on-chain lookups)
- `oddsPairId`: needed for `claimPosition` and `adjustUnmatchedPair` on-chain calls
- `team`: the team the user bet on
- `opponent`: the opposing team
- `market`: `"moneyline"`, `"spread"`, or `"total"`
- `odds`: user's decimal odds (null if unavailable)

**Formatted text:** The `formatted` field returns a Telegram-optimized summary. Shows one-liner per position when counts are small, switches to aggregate counts when there are many.

**Errors:** 400 if address is invalid; returns empty arrays (not 404) if no unclaimed positions

---

## Analytics

### GET /analytics/odds-history/:contestId

Opening lines, current lines, and line movement for a contest. All odds are in decimal format.

```bash
curl "https://api.ospex.org/v1/analytics/odds-history/87"
```

**Response:**

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
  }
}
```

**Errors:** 400 if `contestId` is missing; 404 if contest not found or no odds data

---

### GET /analytics/matchup/:contestId

Comprehensive matchup analysis. Primary: ELO-based win probability. Supplementary sections are fetched in parallel and included when available.

| Name | Type | Default | Constraints |
|------|------|---------|-------------|
| `sport` | string | `ncaab` | `nba`, `nhl`, `ncaab` |

```bash
curl "https://api.ospex.org/v1/analytics/matchup/87?sport=ncaab"
```

**Response:**

```json
{
  "data": {
    "contestId": "87",
    "away": { "teamName": "Kentucky Wildcats", "slug": "kentucky-wildcats", "eloRating": 1731, "nationalRank": 46, "record": "17-8" },
    "home": { "teamName": "Auburn Tigers", "slug": "auburn-tigers", "eloRating": 1633, "nationalRank": 79, "record": "14-11" },
    "prediction": { "awayWinProbability": 0.497, "homeWinProbability": 0.503, "favoredTeam": "Auburn Tigers", "effectiveEloDiff": 2 },
    "schedule": { "away": { ... }, "home": { ... } },
    "standings": { "away": { ... }, "home": { ... } },
    "injuries": { "away": [...], "home": [...] },
    "rankings": { "away": { "apRank": 12, "coachesRank": 14 }, "home": { ... } },
    "expertPicks": [{ "pundit": "...", "org": "...", "pick": "...", "pickType": "winner", "reasoning": "..." }]
  }
}
```

**Supplementary sections:**
- `schedule`: rest days, back-to-back status, recent form. All sports.
- `standings`: record, conference rank, streaks. NBA/NHL only.
- `injuries`: current injury report. NBA/NHL only.
- `rankings`: AP/Coaches poll rank. NCAAB only.
- `expertPicks`: pundit picks if available.
- Sections are silently omitted if data is unavailable — never errors.

**Errors:** 400 if `contestId` is missing or `sport` is invalid; 404 if contest not found or ELO data missing

---

### GET /analytics/schedule/:teamName

Recent and upcoming games, rest days, and back-to-back status.

| Name | Type | Required | Constraints |
|------|------|----------|-------------|
| `sport` | string | **yes** | `nba`, `nhl`, `ncaab` |

```bash
curl "https://api.ospex.org/v1/analytics/schedule/duke-blue-devils?sport=ncaab"
```

**Response:**

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
  }
}
```

**Errors:** 400 if `sport` is missing or invalid; 404 if team not found

---

### GET /analytics/injuries/:teamName

Current injury report. Empty array means the team is healthy. NBA and NHL only.

| Name | Type | Required | Constraints |
|------|------|----------|-------------|
| `sport` | string | **yes** | `nba`, `nhl` |

```bash
curl "https://api.ospex.org/v1/analytics/injuries/los-angeles-lakers?sport=nba"
```

**Response:**

```json
{
  "data": {
    "teamName": "Los Angeles Lakers",
    "sport": "nba",
    "injuries": [
      { "player": "LeBron James", "position": "F", "status": "Day-To-Day", "injuryType": "Knee" }
    ],
    "lastUpdated": "2026-02-21T01:45:12.787+00:00"
  }
}
```

---

### GET /analytics/team-stats/:teamName

Team statistics — point differential, PPG, shooting percentages, special teams. NBA and NHL only.

| Name | Type | Required | Constraints |
|------|------|----------|-------------|
| `sport` | string | **yes** | `nba`, `nhl` |

```bash
curl "https://api.ospex.org/v1/analytics/team-stats/los-angeles-lakers?sport=nba"
```

**Response:**

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
  }
}
```

**Sport-specific fields:**
- **NBA:** `fieldGoalPct`, `threePointPct` (display-ready percentages, e.g., 50.0 = 50.0%)
- **NHL:** `pace` (shots/game), `powerPlayPct`, `penaltyKillPct`, `savePct`

---

### GET /analytics/standings

Full league standings. NBA and NHL only (use `/analytics/rankings` for NCAAB).

| Name | Type | Required | Constraints |
|------|------|----------|-------------|
| `league` | string | **yes** | `nba`, `nhl` |
| `conference` | string | no | Filter by conference (e.g., `Eastern`, `Western`) |

```bash
curl "https://api.ospex.org/v1/analytics/standings?league=nba"
```

**Response:**

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
  }
}
```

**NHL-specific fields:** `otl` (overtime losses), `points` (standings points)

---

### GET /analytics/rankings

NCAAB AP Top 25 or Coaches Poll rankings.

| Name | Type | Default | Constraints |
|------|------|---------|-------------|
| `poll` | string | `ap` | `ap`, `coaches` |

```bash
curl "https://api.ospex.org/v1/analytics/rankings?poll=ap"
```

**Response:**

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
  }
}
```

---

### GET /analytics/rosters/:teamName

Current team roster. NBA and NHL only.

| Name | Type | Required | Constraints |
|------|------|----------|-------------|
| `sport` | string | **yes** | `nba`, `nhl` |

```bash
curl "https://api.ospex.org/v1/analytics/rosters/los-angeles-lakers?sport=nba"
```

**Response:**

```json
{
  "data": {
    "teamName": "Los Angeles Lakers",
    "sport": "nba",
    "roster": [
      { "player": "LeBron James", "position": "F", "jersey": "23", "height": "6' 9\"", "age": 41 }
    ],
    "lastUpdated": "2026-02-20T12:00:00.000+00:00"
  }
}
```

---

### GET /analytics/expert-picks/:contestId

Attributed expert/pundit picks for a contest. Returns empty array if none available.

```bash
curl "https://api.ospex.org/v1/analytics/expert-picks/87"
```

**Response:**

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
  }
}
```

---

### GET /analytics/elo/:teamName

ELO rating for a single team.

| Name | Type | Default | Constraints |
|------|------|---------|-------------|
| `sport` | string | `ncaab` | `nba`, `nhl`, `ncaab` |
| `season` | number | _(current)_ | Range 2000-2100 |

```bash
curl "https://api.ospex.org/v1/analytics/elo/duke?sport=ncaab"
```

**Response:**

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
  }
}
```

**Fields:**
- `nationalRank`: ranking or `null` if unranked
- `trend`: ELO rating change (positive = improving)
- `trendLabel`: `"rising"`, `"falling"`, or `"stable"`
- `slug`: URL-safe team name (use this in requests)

---

### GET /analytics/elo/rankings

Top ELO rankings for a sport.

| Name | Type | Default | Constraints |
|------|------|---------|-------------|
| `sport` | string | `ncaab` | `nba`, `nhl`, `ncaab` |
| `season` | number | _(current)_ | Range 2000-2100 |
| `top` | number | _(all)_ | Range 1-500 |

```bash
curl "https://api.ospex.org/v1/analytics/elo/rankings?sport=ncaab&top=25"
```

**Response:** Array of team ELO objects (same shape as `/analytics/elo/:teamName`).

---

### GET /analytics/elo/all

All ELO ratings for a sport (no top limit).

| Name | Type | Default | Constraints |
|------|------|---------|-------------|
| `sport` | string | `ncaab` | `nba`, `nhl`, `ncaab` |
| `season` | number | _(current)_ | Range 2000-2100 |

**Response:** Same array format as `/analytics/elo/rankings`.

---

## Protocol

### GET /protocol/info

Static protocol metadata — network, contracts, supported sports, fees.

```bash
curl "https://api.ospex.org/v1/protocol/info"
```

**Response:**

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
  }
}
```

---

### GET /protocol/agents

Known automated agents with descriptions and last run times.

```bash
curl "https://api.ospex.org/v1/protocol/agents"
```

**Response:**

```json
{
  "data": {
    "agents": [
      { "agentId": "market_maker_michelle", "network": "polygon", "lastRun": "2026-02-21T14:15:43.430Z", "description": "Market maker agent. Provides liquidity by posting positions on both sides." },
      { "agentId": "degen_dan", "network": "polygon", "lastRun": "2026-02-21T19:14:59.613Z", "description": "Aggressive bettor agent. Takes positions on favorites." }
    ]
  }
}
```

---

### GET /leaderboard

Current active leaderboard standings.

| Name | Type | Default | Constraints |
|------|------|---------|-------------|
| `top` | number | 50 | Range 1-200 |

```bash
curl "https://api.ospex.org/v1/leaderboard?top=10"
```

**Response:**

```json
{
  "data": {
    "leaderboardId": "7",
    "name": "February 2026",
    "startTime": "2026-02-01T00:00:00.000Z",
    "endTime": "2026-02-28T23:59:59.000Z",
    "entries": [
      { "rank": 1, "address": "0xabcd...1234", "declaredBankroll": 250.0 }
    ]
  }
}
```

Returns `{ "data": null, "formatted": "No active leaderboard found." }` if no leaderboard is active.

---

## Instant Match

The instant match system connects users with Michelle (the automated market maker agent). The flow is: request a quote → optionally accept a counter-offer → create position on-chain → call match.

### POST /instant-match/:speculationId/quote

Request a quote for an existing on-chain speculation. Default response is Server-Sent Events (SSE). Add `?stream=false` for a single JSON response.

**Request body:**

```json
{
  "side": "away",
  "amountUSDC": 3.0,
  "odds": 1.85,
  "oddsFormat": "decimal",
  "wallet": "0xYourWalletAddress"
}
```

**Fields:**
- `side`: `"over"`, `"under"`, `"home"`, or `"away"`
- `amountUSDC`: position size in USDC (must be > 0)
- `odds`: requested odds in the format specified by `oddsFormat`
- `oddsFormat`: `"decimal"` (default), `"american"`, or `"probability"`
- `wallet`: your wallet address (0x-prefixed)

#### SSE response (default)

Stream events: `progress` (intermediate updates), `result` (final outcome), `error` (evaluation failed). Keepalive comments sent every 15s.

**Approved result:**

```json
{
  "approved": true,
  "quoteId": "quote_abc123",
  "approvedOddsDecimal": 1.85,
  "approvedOddsAmerican": -118,
  "expiresAt": "2026-02-22T00:01:00.000Z"
}
```

**Counter-offer result** (approved but at different odds):

```json
{
  "approved": true,
  "quoteId": "quote_abc123",
  "approvedOddsDecimal": 1.90,
  "approvedOddsAmerican": -111,
  "counterOffer": { "oddsDecimal": 1.90, "oddsAmerican": -111, "ttlSeconds": 60 },
  "expiresAt": "2026-02-22T00:01:00.000Z"
}
```

**Rejected result:**

```json
{ "approved": false, "reason": "Odds too far from market" }
```

#### Sync response (`?stream=false`)

Returns a single JSON object with the same shape as the SSE `result` event. HTTP 200 is committed before evaluation starts; errors appear in the response body. 90-second timeout.

```bash
curl -X POST "https://api.ospex.org/v1/instant-match/101/quote?stream=false" \
  -H "Content-Type: application/json" \
  -d '{"side":"away","amountUSDC":3,"odds":1.85,"oddsFormat":"decimal","wallet":"0xYourWallet"}'
```

**Pre-stream errors:**

| Code | HTTP | Meaning |
|------|------|---------|
| `INVALID_REQUEST` | 400 | Missing or invalid field in request body |
| `INVALID_ODDS` | 400 | Odds conversion failed or result <= 1.0 |
| `SPECULATION_NOT_FOUND` | 404 | Speculation does not exist |
| `INVALID_SPECULATION` | 400 | Unknown market type for the speculation |

**Stream/evaluation errors:**

| Code | Meaning |
|------|---------|
| `NOT_AVAILABLE` | Market maker is not available for this market |
| `BUSY` | Market maker is evaluating another request |
| `QUOTE_CREATION_FAILED` | Internal error creating the quote |
| `EVALUATION_ERROR` | LLM evaluation failed |

---

### POST /instant-match/quote

Request a quote without an existing speculation (contest-path variant). Same response format as above (SSE default, `?stream=false` for sync). The market type and contest are specified in the request body.

**Request body:**

```json
{
  "contestId": 42,
  "marketType": "moneyline",
  "side": "away",
  "amountUSDC": 3.0,
  "odds": 1.85,
  "oddsFormat": "decimal",
  "wallet": "0xYourWalletAddress"
}
```

**Additional fields (required for this variant):**
- `contestId`: on-chain contest ID (number)
- `marketType`: `"moneyline"`, `"spread"`, or `"total"`
- `line`: required for spread/total markets; omit for moneyline

**Additional errors:**

| Code | HTTP | Meaning |
|------|------|---------|
| `INVALID_REQUEST` | 400 | Missing `contestId` or `marketType` |
| `INVALID_REQUEST` | 400 | Missing `line` for spread/total markets |
| `NOT_FOUND` | 404 | Contest not found or missing odds reference data |

---

### POST /instant-match/:quoteId/accept-counter

Accept a counter-offer from the market maker. Required before calling the match endpoint when the quote response included a `counterOffer` object.

**Request body:**

```json
{ "wallet": "0xYourWalletAddress" }
```

**Response (200):**

```json
{
  "accepted": true,
  "quoteId": "quote_abc123",
  "approvedOddsDecimal": 1.92,
  "approvedOddsAmerican": -110,
  "expiresAt": "2026-03-04T15:35:00.000Z"
}
```

**Errors:**

| Code | HTTP | Meaning |
|------|------|---------|
| `INVALID_REQUEST` | 400 | Missing `quoteId` or `wallet` |
| `QUOTE_NOT_FOUND` | 404 | Quote does not exist |
| `INVALID_QUOTE_STATE` | 400 | Quote is not in `counter` state |
| `WALLET_MISMATCH` | 403 | Wallet does not match the original quote |
| `COUNTER_EXPIRED` | 410 | Counter-offer TTL has elapsed (default 60s) |

---

### POST /instant-match/:quoteId/match

Execute a match against an approved quote. Call this after creating the position on-chain.

**Request body:**

```json
{ "positionId": "101_0xabcd..._42_0" }
```

**Response (200):**

```json
{ "matched": true, "txHash": "0x..." }
```

**Errors:**

| Code | HTTP | Meaning |
|------|------|---------|
| `INVALID_REQUEST` | 400 | Missing or invalid `positionId` |
| `QUOTE_NOT_FOUND` | 404 | Quote does not exist |
| `QUOTE_EXPIRED` | 410 | Quote TTL has elapsed |
| `INVALID_QUOTE_STATE` | 400 | Quote is not in a matchable state |
| `POSITION_NOT_FOUND` | 400 | Position does not exist on-chain yet |
| `POSITION_MISMATCH` | 400 | Position does not match the quote parameters |
| `WALLET_MISMATCH` | 403 | Position wallet does not match quote wallet |
| `STAKE_TOO_SMALL` | 400 | Position stake is below the minimum |
| `MATCH_EXECUTION_FAILED` | 500 | On-chain match transaction failed |
