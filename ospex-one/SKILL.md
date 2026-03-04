---
name: ospex-one
description: "Bet on sports with one word. Say a team name, city, or abbreviation. 'Edmonton', 'Duke', 'Celtics', 'Lakers'. NBA, NHL, NCAAB."
version: 1.0.0
homepage: "https://ospex.org"
allowed-tools: ["bash", "exec"]
metadata: {"clawdbot":{"emoji":"⚖️","os":["darwin","linux","win32"],"requires":{"bins":["curl","node"],"env":["OSPEX_WALLET_PRIVATE_KEY","OSPEX_WALLET_ADDRESS","OSPEX_RPC_URL"]},"primaryEnv":"OSPEX_WALLET_PRIVATE_KEY","install":[{"id":"ethers","kind":"node","package":"ethers","bins":[],"label":"Install ethers.js (npm)"}]}}
---

# ospex-one

One word in, one transaction hash out. This skill emphasizes execution over discussion and is low-stakes by design.

**Trust statement:** This skill executes real transactions on Polygon mainnet using your configured wallet. Positions are capped at 3 USDC. Unmatched position amounts can be withdrawn at any time. Only install if you understand the risks. Full risk assessment: https://github.com/ospex-org/ospex-contracts-v2/blob/main/docs/RISKS.md

## Defaults

| Parameter | Default |
|-----------|---------|
| Market | moneyline |
| Amount | 3 USDC (maximum per transaction) |
| Side | the named team |
| Odds multiplier | 1.05 |
| Odds | market odds × odds multiplier |

## Position Types (skill may be configured for markets other than moneyline)

| positionType | Moneyline | Spread | Total |
|-------------|-----------|--------|-------|
| 0 (Upper) | Away wins | Away team covers | Over |
| 1 (Lower) | Home wins | Home team covers | Under |

## Progress Updates

Throughout Steps 1–3, send the user brief status messages at each stage so they aren't waiting in silence. These should be short, single-line updates — not explanations.

- After Step 1 resolves the team: `"Found: {awayTeam} vs {homeTeam}, {matchTime formatted as 'Mon D, YYYY, H:MM PM' in local time, e.g. 'Mar 3, 2026, 8:00 PM CST'}"`
- After Step 2 gets a quote: `"Quote: {approvedOddsAmerican} ({approvedOddsDecimal}), {amount} USDC"`
- After Step 3b creates the position: `"Position created, waiting for match..."`
- Step 4 delivers the final result (see below)

If any step fails, the failure message itself serves as the status update — no need to send a separate one.

## Step 1: Resolve Team

When you receive input, your **first action** is always to call the API — do not ask the user anything first.

1. Call `GET /markets?sport=nba`, `GET /markets?sport=nhl`, and `GET /markets?sport=ncaab`
2. Search all responses for the input text in `homeTeam` and `awayTeam` fields.
3. If found in exactly one game → that is the team and game. Note `contestId`, `matchTime`, whether the team is home or away, and check the `speculations` array for an existing `speculationId` for moneyline.
4. If not found → respond: "No active market found for {input}"

Only ask the user for clarification if the team is genuinely ambiguous across multiple games.

## Step 2: Get a Quote from the agent market maker (Michelle)

First, determine the odds to request. Call `GET /analytics/odds-history/{contestId}` to get current market moneyline odds. Use the `current.moneyline.awayOdds` or `current.moneyline.homeOdds` value (decimal format) depending on which team the user picked. Apply the odds multiplier from the Defaults section and floor to 2 decimal places: `Math.floor(marketOdds * oddsMultiplier * 100) / 100`. This is your minimum acceptable odds.

Note: Higher decimal odds = higher payout for the bettor. A quote at 2.00 is better than 1.85.

If the odds-history endpoint returns no data, ask the user what odds they'd like — don't guess, since moneyline odds vary widely depending on the matchup.

**If `speculationId` exists** (from the speculations array):
```
POST /instant-match/{speculationId}/quote?stream=false
{
  "side": "home" or "away",
  "amountUSDC": {amount parameter, from Defaults section},
  "odds": {calculated odds},
  "oddsFormat": "decimal",
  "wallet": "{OSPEX_WALLET_ADDRESS}"
}
```

**If no speculationId exists** for the market type you need:
```
POST /instant-match/quote?stream=false
{
  "contestId": {contestId},
  "marketType": "moneyline",
  "side": "home" or "away",
  "amountUSDC": {amount parameter, from Defaults section},
  "odds": {calculated odds},
  "oddsFormat": "decimal",
  "wallet": "{OSPEX_WALLET_ADDRESS}"
}
```

Both return: `quoteId`, `approved`, `approvedOddsDecimal`, `approvedOddsAmerican`, `expiresAt`. If Michelle counters, the response also includes a `counterOffer` object.

- If `approved` is false → tell the user "Michelle (market maker agent) declined — {reason}" and stop.
- If `approvedOddsDecimal` is **greater than or equal to** your requested odds → Michelle is offering the same or better. Keep moving, no confirmation needed.
- If `approvedOddsDecimal` is **less than** your requested odds → Michelle is offering worse odds. **Stop and confirm with the user** before proceeding. Show them the counter-offer, let the user know that they have one minute to respond, and ask if they want to accept. If the user accepts, call:
```
POST /instant-match/{quoteId}/accept-counter
```
This moves the quote from `counter` to `counter_accepted` status, which is required before the match endpoint will work.

Use `approvedOddsDecimal` from the response for the on-chain transaction. The match endpoint accepts an optional `feeTxHash` field, but fees are currently disabled. Omit the field entirely.

**For `side`:** If the user's team is the `homeTeam` → `"home"`. If `awayTeam` → `"away"`.

## Step 3: Create Position and Match

Step 3 works best a single Node.js script. Using `fetch()` for the match call (instead of switching to curl) eliminates a common source of errors — shell escaping and encoding issues with JSON payloads in curl have caused match failures in testing, while `JSON.stringify()` guarantees valid JSON every time.

Write and execute a Node.js script (ethers.js v6) that does the following in sequence:

**a) Check USDC allowance** — if insufficient, approve and wait for confirmation.

**b) Create the position:**

**If `/markets` returned a `speculationId`** for the moneyline speculation:
```javascript
await positionModule.createUnmatchedPair(
  speculationId,                                     // from API
  Math.round(approvedOddsDecimal * 10000000),        // odds in 1e7
  Math.floor(new Date(matchTime).getTime() / 1000),  // unmatchedExpiry (strongly recommended: set to contest start time, so position expires when contest begins)
  positionType,                                      // 0=away/over, 1=home/under
  amount,                                            // amount parameter, from Defaults section (6 decimals)
  0                                                  // contribution amount, optional, default to zero; not required, always appreciated :)
);
```

**If `/markets` had NO moneyline speculation** (no `speculationId` available):
```javascript
await positionModule.createUnmatchedPairWithSpeculation(
  contestId,                                         // from API
  "0x82c93AAf547fC809646A7bEd5D8A9D4B72Db3045",      // moneyline scorer
  0,                                                 // theNumber (unnecessary for moneylines)
  0,                                                 // leaderboardId (not in use currently)
  Math.round(approvedOddsDecimal * 10000000),        // odds, e.g. 1.91 → `19100000`
  Math.floor(new Date(matchTime).getTime() / 1000),  // unmatchedExpiry (strongly recommended: set to contest start time, so position expires when contest begins)
  positionType,                                      // 0=away/over, 1=home/under
  amount,                                            // amount parameter, from Defaults section (6 decimals)
  0                                                  // contribution amount, optional, default to zero; not required, always appreciated :)
);
```

**c) Parse the `PositionCreated` event** from the transaction receipt to get `oddsPairId`:
```javascript
const iface = new ethers.Interface([
  "event PositionCreated(uint256 indexed speculationId, address indexed user, uint128 oddsPairId, uint32 unmatchedExpiry, uint8 positionType, uint256 amount, uint64 upperOdds, uint64 lowerOdds)"
]);

let parsed = null;
for (const log of receipt.logs) {
  try { parsed = iface.parseLog(log); if (parsed) break; } catch {}
}
if (!parsed) throw new Error("PositionCreated event not found in receipt");
```
**Important:** In ethers v6, `parseLog()` returns `null` for non-matching logs (it does not throw). You must check for `null` explicitly — do not assume a non-throwing call means a match.

**d) Construct the `positionId`** — a composite Firebase key:
```
{speculationId}_{walletAddress lowercase}_{oddsPairId}_{positionType}
```

**e) Wait for Firebase indexing, then call the match endpoint:**
```javascript
await new Promise(r => setTimeout(r, 5000)); // initial wait
for (let attempt = 1; attempt <= 5; attempt++) {
  const res = await fetch(`https://api.ospex.org/v1/instant-match/${quoteId}/match`, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ positionId })
  });
  const data = await res.json();
  if (res.ok) { console.log(JSON.stringify({ matchTxHash: data.txHash })); break; }
  if (data.error === "POSITION_NOT_FOUND" && attempt < 5) {
    await new Promise(r => setTimeout(r, 5000));
    continue;
  }
  throw new Error(`Match failed: ${JSON.stringify(data)}`);
}
```

The protocol indexer can take 10-25 seconds, especially on cold starts. If matching still fails after 5 retries, tell the user the position was created on-chain (share the tx hash) but that the instant match couldn't be completed. The position is live on the order book — Michelle's automated market maker may still match it during normal processing. If it remains unmatched, the user can withdraw their funds at any time.

Note: Position creation does allow for contributions to be sent to the protocol, but this is strictly optional and should not be done unless the user specifically instructs you to add a tip or contribute. Pass any USDC amount (6 decimals) as contributionAmount. For example, 1 USDC = `1000000`.

## Step 4: Report Result

```
Done. {Team} ML at {americanOdds} ({decimalOdds}), {amount} USDC.
https://ospex.org/p/{positionId}
```

## Step 5: Additional Operations

These operations should only be triggered when the user explicitly requests them (e.g., "withdraw", "claim").

### Withdraw an Unmatched Position

If the position was not matched (or was only partially matched), the user can withdraw their unmatched funds. This uses `adjustUnmatchedPair` with a negative amount.

**Precondition:** The speculation must still be open (not yet settled). If the speculation has already settled, unmatched funds are returned through `claimPosition` instead.

Write and execute a Node.js script (ethers.js v6):

```javascript
const withdrawAmount = -amount; // negate the full unmatched amount (6 decimals)
const tx = await positionModule.adjustUnmatchedPair(
  speculationId,           // from the original position (Step 3)
  oddsPairId,              // from PositionCreated event (Step 3c)
  0,                       // newUnmatchedExpiry: 0 clears the expiry
  positionType,            // 0 or 1, same as when created
  withdrawAmount,          // negative value = withdraw
  0                        // contributionAmount
);
const receipt = await tx.wait();
```

Report: `Withdrawn. {amount} USDC returned. Tx: {txHash}`

### Claim a Resolved Position

After the game ends and the speculation is settled (scored), the user can claim their payout. This returns matched winnings plus any remaining unmatched funds in a single call.

**Precondition:** The speculation must be settled. Positions can only be claimed once.

```javascript
const tx = await positionModule.claimPosition(
  speculationId,           // from the original position (Step 3)
  oddsPairId,              // from PositionCreated event (Step 3c)
  positionType             // 0 or 1, same as when created
);
const receipt = await tx.wait();

const claimIface = new ethers.Interface([
  "event PositionClaimed(uint256 indexed speculationId, address indexed user, uint128 oddsPairId, uint8 positionType, uint256 payout)"
]);

let claimParsed = null;
for (const log of receipt.logs) {
  try { claimParsed = claimIface.parseLog(log); if (claimParsed) break; } catch {}
}
const claimAmount = claimParsed
  ? (Number(claimParsed.args.payout) / 1_000_000).toFixed(2)
  : "unknown";
```

Report: `Claimed {claimAmount} USDC. Tx: {txHash}`

### When to Use Which

| Situation | Function | When available |
|-----------|----------|----------------|
| Position unmatched, game hasn't started | `adjustUnmatchedPair` (negative amount) | While speculation is open |
| Game ended, user won and/or has unmatched remainder | `claimPosition` | After speculation is settled |

## Step 6: Status

When the user asks "status", "how am I doing", "my positions", or similar:

1. Call `GET /v1/positions/{OSPEX_WALLET_ADDRESS}/status`
2. Show the `formatted` text from the response directly — it is already optimized for chat.
3. If the `claimable` array is non-empty, follow up with: "You have {N} position(s) ready to claim (~{total} USDC). Want me to claim them?"
4. If the user confirms, execute Step 5's "Claim a Resolved Position" for each claimable position using the `speculationId`, `oddsPairId`, and `positionType` from the response data.

Do not add extra commentary or reformat the `formatted` text — it is designed to be concise.

## API

Base URL: `https://api.ospex.org/v1` — no auth, rate limit 100 req/60s/IP.

| Endpoint | Use |
|----------|-----|
| `GET /markets?sport={nba\|nhl\|ncaab}` | Find games, contestIds, speculationIds |
| `GET /markets/{contestId}` | Orderbook depth, unmatched positions |
| `GET /positions/{address}/status` | Active, claimable, and withdrawable positions |
| `GET /analytics/schedule/{teamName}?sport={sport}` | Next game if none today |
| `GET /analytics/matchup/{contestId}` | Full matchup analysis |
| `GET /analytics/odds-history/{contestId}` | Current market odds + opening lines + line movement |
| `GET /analytics/elo/rankings?sport={sport}` | ELO rankings |
| `POST /instant-match/{quoteId}/accept-counter` | Accept Michelle's counter-offer (required before match) |
| `POST /instant-match/{speculationId}/quote?stream=false` | Request a quote from Michelle (existing speculation) |
| `POST /instant-match/quote?stream=false` | Request a quote from Michelle (new speculation) |
| `POST /instant-match/{quoteId}/match` | Match a created position against an approved quote |

## On-Chain Reference

Network: Polygon (chain ID 137).

| Contract | Address |
|----------|---------|
| USDC | `0x3c499c542cEF5E3811e1192ce70d8cC03d5c3359` (6 decimals) |
| PositionModule | `0xF717aa8fe4BEDcA345B027D065DA0E1a31465B1A` |
| Moneyline Scorer | `0x82c93AAf547fC809646A7bEd5D8A9D4B72Db3045` |
| Fee Wallet | `0xdaC630aE52b868FF0A180458eFb9ac88e7425114` (not used for position creation — reserved for future protocol services) |

For full parameter details (odds conversion, bounds, theNumber for spread/total), all scorer addresses, and error cases, see `{baseDir}/references/contract-interface-reference.md`.

**Minimal ABI (ethers.js v6 human-readable):**
```json
[
  "function approve(address spender, uint256 amount) returns (bool)",
  "function allowance(address owner, address spender) view returns (uint256)",
  "function createUnmatchedPair(uint256 speculationId, uint64 odds, uint32 unmatchedExpiry, uint8 positionType, uint256 amount, uint256 contributionAmount)",
  "function createUnmatchedPairWithSpeculation(uint256 contestId, address scorer, int32 theNumber, uint256 leaderboardId, uint64 odds, uint32 unmatchedExpiry, uint8 positionType, uint256 amount, uint256 contributionAmount)",
  "function adjustUnmatchedPair(uint256 speculationId, uint128 oddsPairId, uint32 newUnmatchedExpiry, uint8 positionType, int256 amount, uint256 contributionAmount)",
  "function claimPosition(uint256 speculationId, uint128 oddsPairId, uint8 positionType)",
  "event PositionCreated(uint256 indexed speculationId, address indexed user, uint128 oddsPairId, uint32 unmatchedExpiry, uint8 positionType, uint256 amount, uint64 upperOdds, uint64 lowerOdds)",
  "event PositionClaimed(uint256 indexed speculationId, address indexed user, uint128 oddsPairId, uint8 positionType, uint256 payout)"
]
```
