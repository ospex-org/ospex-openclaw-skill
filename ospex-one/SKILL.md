---
name: ospex-one
description: "Bet on sports with one word. Say a team name, city, or abbreviation. 'Edmonton', 'Duke', 'Celtics', 'Lakers'. NBA, NHL, NCAAB."
version: 1.2.0
homepage: "https://ospex.org"
allowed-tools: ["bash", "exec"]
metadata: {"clawdbot":{"emoji":"⚖️","os":["darwin","linux","win32"],"requires":{"bins":["curl","node"],"env":["OSPEX_WALLET_PRIVATE_KEY","OSPEX_WALLET_ADDRESS","OSPEX_RPC_URL"]},"primaryEnv":"OSPEX_WALLET_PRIVATE_KEY","install":[{"id":"ethers","kind":"node","package":"ethers","bins":[],"label":"Install ethers.js (npm)"}]}}
---

# ospex-one

One word in, one transaction hash out. This skill emphasizes execution over discussion and is low-stakes by design.

**Trust statement:** This skill executes real transactions on Polygon mainnet using your configured wallet. Positions are capped at 3 USDC. Unmatched position amounts can be withdrawn at any time. Only install if you understand the risks. Full risk assessment: https://github.com/ospex-org/ospex-contracts-v2/blob/main/docs/RISKS.md

**Input expectations:** This skill is designed for single-word input — a team name, city, or abbreviation. If the user's message contains additional instructions, modifiers, or betting language beyond a team name (e.g., "lay the points", "take the under", "bet $5", "Celtics -6.5"), act on it if the intent is unambiguous — otherwise, ask the user to clarify before proceeding.

## Defaults

| Parameter | Default |
|-----------|---------|
| Market | moneyline |
| Amount | 3 USDC (maximum per transaction) |
| Side | the named team |
| Odds multiplier | 1.05 |
| Odds | market odds × odds multiplier |

## Progress Updates

Throughout Steps 1–3, send the user brief status messages at each stage so they aren't waiting in silence. These should be short, single-line updates — not explanations.

- After Step 1 resolves the team: `"Found: {awayTeam} vs {homeTeam}, {matchTime formatted as 'Mon D, YYYY, H:MM PM' in EST, e.g. 'Mar 3, 2026, 8:00 PM EST'}"`
- After Step 2 gets a quote: `"Quote: {approvedOddsAmerican} ({approvedOddsDecimal}), {amount} USDC"`
- After Step 3 creates the position: `"Position created, waiting for match..."`
- Step 4 delivers the final result

If any step fails, the failure message itself serves as the status update — no need to send a separate one. If any API call returns an unexpected error, stop and report it. Do not silently retry or work around failures.

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

**Save the `txParams` object from the response — you will need it in Step 3.** This contains all pre-computed on-chain transaction parameters. Do not compute these values yourself.

- If `approved` is false → tell the user "Michelle (market maker agent) declined — {reason}" and stop.
- If `approvedOddsDecimal` is **greater than or equal to** your requested odds → Michelle is offering the same or better. Keep moving, no confirmation needed.
- If `approvedOddsDecimal` is **less than** your requested odds → Michelle is offering worse odds. **Stop and confirm with the user** before proceeding. Show them the counter-offer, let the user know that they have one minute to respond, and ask if they want to accept. If the user accepts, call:
```
POST /instant-match/{quoteId}/accept-counter
Body: { "wallet": "{OSPEX_WALLET_ADDRESS}" }
```
This returns an updated `txParams` object. **Use the txParams from this response** (it reflects the accepted counter-offer terms).

**For `side`:** If the user's team is the `homeTeam` → `"home"`. If `awayTeam` → `"away"`.

## Step 3: Create Position and Match

Write and execute a **single Node.js script** (ethers.js v6) that does everything below in sequence. Do not break this into multiple script executions — the entire flow must run in one script to avoid unnecessary latency.

The `txParams` object from Step 2 tells you exactly which contract method to call and with what arguments. Use it directly.

**a) Check USDC allowance** — if insufficient, approve and wait for confirmation.

**b) Create the position using `txParams`:**

`txParams.method` is either `"createUnmatchedPair"` or `"createUnmatchedPairWithSpeculation"`. Call the corresponding contract function, passing the values from `txParams.args` directly. Do not compute odds, timestamps, positionType, or any other parameter yourself — use the values from txParams as-is.

```javascript
// txParams.method === "createUnmatchedPair"
const tx = await positionModule.createUnmatchedPair(
  txParams.args.speculationId,
  txParams.args.odds,
  txParams.args.unmatchedExpiry,
  txParams.args.positionType,
  txParams.args.amount,
  txParams.args.contributionAmount
);

// OR txParams.method === "createUnmatchedPairWithSpeculation"
const tx = await positionModule.createUnmatchedPairWithSpeculation(
  txParams.args.contestId,
  txParams.args.scorer,
  txParams.args.theNumber,
  txParams.args.leaderboardId,
  txParams.args.odds,
  txParams.args.unmatchedExpiry,
  txParams.args.positionType,
  txParams.args.amount,
  txParams.args.contributionAmount
);
```
Contributions: `txParams.args.contributionAmount` defaults to "0". Contributions are optional tips sent to the protocol alongside a position. If the user explicitly asks to tip or contribute (e.g., "Lakers, tip 1 USDC"), replace this value with the USDC amount scaled to 6 decimals (1 USDC = "1000000"). Never add a contribution unless the user specifically requests it.

Wait for the transaction to be mined: `const receipt = await tx.wait();`

**c) Get the positionId from the API:**

Do **not** parse the transaction receipt yourself. Call the server-side endpoint which does this deterministically:

```javascript
const posRes = await fetch(`https://api.ospex.org/v1/positions/by-tx/${receipt.hash}`);
const posData = await posRes.json();
const positionId = posData.positions[0].positionId;
```

**d) Call the match endpoint:**

```javascript
await new Promise(r => setTimeout(r, 5000)); // wait for Firebase indexing
for (let attempt = 1; attempt <= 5; attempt++) {
  const res = await fetch(`https://api.ospex.org/v1/instant-match/${quoteId}/match`, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ positionId })
  });
  const data = await res.json();
  if (res.ok) { console.log(JSON.stringify({ matchTxHash: data.txHash, positionId })); break; }
  if (data.code === "POSITION_NOT_FOUND" && attempt < 5) {
    await new Promise(r => setTimeout(r, 5000));
    continue;
  }
  throw new Error(`Match failed: ${JSON.stringify(data)}`);
}
```

The protocol indexer can take 10-25 seconds, especially on cold starts. If matching still fails after 5 retries, tell the user the position was created on-chain (share the tx hash) but that the instant match couldn't be completed. The position is live on the order book — Michelle's automated market maker may still match it during normal processing. If it remains unmatched, the user can withdraw their funds at any time.

## Step 4: Report Result

```
Done. {Team} ML at {americanOdds} ({decimalOdds}), {amount} USDC.
https://ospex.org/p/{positionId}
```

## Step 5: Additional Operations

These operations should only be triggered when the user explicitly requests them (e.g., "status", "withdraw", "claim").

### Status check

When the user asks "status", "how am I doing", or similar:

1. Call `GET /positions/{OSPEX_WALLET_ADDRESS}/status`
2. Show the `formatted` text from the response directly — it is already optimized for chat.
3. If the `claimable` array is non-empty, follow up with: "You have {N} position(s) ready to claim (~{total} USDC). Want me to claim them?"
4. If the user confirms, follow the "Claim" flow below.

### Withdraw an Unmatched Position

If the position was not matched (or was only partially matched), the user can withdraw their unmatched funds.

**Precondition:** The speculation must still be open (not yet settled). If the speculation has already settled, unmatched funds are returned through claiming instead.

1. Call `GET /positions/{OSPEX_WALLET_ADDRESS}/withdraw-params`
2. The response contains a `positions` array. Each entry includes a `description` (e.g., "Lakers ML — Unmatched, 3.00 USDC") and a `txParams` object.
3. Write and execute a Node.js script (ethers.js v6) that calls `positionModule.adjustUnmatchedPair()` using the values from `txParams.args` directly. Do not compute any arguments yourself. Wait for the transaction to be mined.
4. Call `GET /positions/withdraw-result/{txHash}` to get the confirmed amount returned.

Report: `Withdrawn. {amountReturned} USDC returned. Tx: {txHash}`

### Claim a Resolved Position

After the game ends and the speculation is settled (scored), the user can claim their payout. This returns matched winnings plus any remaining unmatched funds in a single call.

**Precondition:** The speculation must be settled. Positions can only be claimed once.

1. Call `GET /positions/{OSPEX_WALLET_ADDRESS}/claim-params`
2. The response contains a `positions` array. Each entry includes a `description` (e.g., "Celtics ML — Won") and a `txParams` object.
3. Write and execute a single Node.js script (ethers.js v6) that calls `positionModule.claimPosition()` for each position, using the values from each entry's `txParams.args` directly. Do not compute any arguments yourself. Wait for each transaction to be mined.
4. Call `GET /positions/claim-result/{txHash}` to get the confirmed payout amount.

Report: `Claimed {payout} USDC. Tx: {txHash}`

### When to Use Which

| Situation | Function | When available |
|-----------|----------|----------------|
| Position unmatched, game hasn't started | `adjustUnmatchedPair` (negative amount) | While speculation is open |
| Game ended, user won and/or has unmatched remainder | `claimPosition` | After speculation is settled |

## API

Base URL: `https://api.ospex.org/v1` — no auth, rate limit 100 req/60s/IP.

| Endpoint | Use |
|----------|-----|
| `GET /markets?sport={nba\|nhl\|ncaab}` | Find games, contestIds, speculationIds |
| `GET /positions/{address}/status` | Active, claimable, and withdrawable positions |
| `GET /positions/by-tx/{txHash}` | Get positionId from a transaction hash (server-side event parsing) |
| `GET /positions/{address}/claim-params` | Pre-computed txParams for all claimable positions |
| `GET /positions/{address}/withdraw-params` | Pre-computed txParams for all withdrawable positions |
| `GET /positions/claim-result/{txHash}` | Parse claim receipt, return payout amount |
| `GET /positions/withdraw-result/{txHash}` | Parse withdraw receipt, return amount returned |
| `GET /analytics/odds-history/{contestId}` | Current market odds + opening lines + line movement |
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

For the complete API (including endpoints not used by this skill), see `{baseDir}/references/api-reference.md`. For contract parameter details (odds conversion, bounds, theNumber for spread/total), all scorer addresses, and error cases, see `{baseDir}/references/contract-interface-reference.md`.

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
