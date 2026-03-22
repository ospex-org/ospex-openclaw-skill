---
name: ospex
description: "Place sports bets on Ospex, an agent-supported peer-to-peer protocol. Supports NBA, NHL, and NCAAB."
version: 1.0.0
homepage: "https://ospex.org"
allowed-tools: ["bash", "exec"]
compatibility: "Requires Node and ethers.js v6. Required env: OSPEX_WALLET_PRIVATE_KEY (wallet private key; high-sensitivity — do not log or expose), OSPEX_WALLET_ADDRESS, OSPEX_RPC_URL (Polygon RPC). Use a dedicated low-fund betting wallet; do not use your primary wallet."
metadata: {"clawdbot":{"emoji":"⚖️","os":["darwin","linux","win32"],"requires":{"bins":["node"],"env":["OSPEX_WALLET_PRIVATE_KEY","OSPEX_WALLET_ADDRESS","OSPEX_RPC_URL"]},"primaryEnv":"OSPEX_WALLET_PRIVATE_KEY","install":[{"id":"ethers","kind":"node","package":"ethers","bins":[],"label":"Install ethers.js (npm)"}]}}
---

# ospex

MCP-native skill for placing sports bets on Ospex. All market data and quote management flows through MCP tools. Node.js scripts handle on-chain transaction signing (the MCP server cannot sign transactions).

**Invocation:** This skill is **invocation-only**: the agent will not use it unless you explicitly ask or reference a team name in context. That avoids accidental bets.

**Trust & safety:** This skill executes real transactions on Polygon mainnet using your configured wallet. Positions are capped at 3 USDC. Unmatched position amounts can be withdrawn at any time. The agent verifies all transaction parameters against hardcoded contract addresses and expected method names before signing — if anything is unexpected, it halts and reports rather than proceeding. OSPEX_WALLET_PRIVATE_KEY is high-sensitivity: the agent must never log, echo, or expose it. Only use this skill with a dedicated low-fund wallet — do not configure it with a wallet containing more funds than you are willing to lose. Full risk assessment: https://github.com/ospex-org/ospex-contracts-v2/blob/main/docs/RISKS.md

## MCP Connection

Server: `POST https://api.ospex.org/mcp` — Streamable HTTP, JSON-RPC, stateless. No auth required for public tools.

**Market Data:** `get-markets`, `get-market-detail`, `get-full-schedule`, `search-current-odds`, `get-current-odds-by-id`
**Analytics:** `get-matchup-analysis`, `get-odds-history`
**Quoting:** `get-instant-match-quote`, `accept-counter-offer`
**Execution:** `execute-instant-match` (requires API key — see Step 4 fallback)
**Positions:** `get-position-status`, `get-position-by-tx`, `get-claim-params`, `get-withdraw-params`, `get-claim-result`, `get-withdraw-result`
**Protocol:** `get-protocol-info`, `get-leaderboard`

For full endpoint documentation beyond what MCP tools provide, load `{baseDir}/references/api-reference.md`.

## Defaults

| Parameter | Default |
|-----------|---------|
| Market | moneyline (override with spread/total via input like "Lakers -6.5" or "over 220.5") |
| Amount | 3 USDC (maximum per transaction) |
| Side | the named team |
| Odds multiplier | 1.05 |
| Odds | market odds x odds multiplier |

## Communication Suggestions

If any MCP tool or API call returns an unexpected error, stop and report it. Do not silently retry or work around failures.

Message the user when one of these occurs:

- **Ambiguity:** The team name matches multiple games and you need clarification.
- **Counter-offer:** Michelle offered worse odds and you need the user's approval before proceeding.
- **Final result:** The position is created and matched (or the match fallback message).
- **Hard failure:** An API error, on-chain revert, or any condition that stops the flow.

## Step 1: Resolve Team and Market Type

When you receive input, your **first action** is always to call `get-markets`. Accept team names, cities, abbreviations, or natural language betting expressions (e.g., "Lakers -6.5", "take the under in the Celtics game").

**Detect market type from input:**
- Point value (e.g., "Atlanta -6.5") → **spread**, line is the number with its sign
- "over"/"under" with a number (e.g., "Lakers over 220.5") → **total**, line is the number
- Otherwise → **moneyline** (default)

**For spread and total:** If the user specifies a line differing from the current market, inform them and ask if they want to proceed at the market line instead.

Search the `data` array for the input text in `homeTeam` and `awayTeam` fields. Never use `formatted` for branching — always use `data`.

If found in exactly one game → note `contestId`, `matchTime`, and whether the team is home or away. **Check the `speculations` array for the detected market type:**

- **If speculationId exists** → note the `speculationId`. For spread, note `awayLine`/`homeLine` (use whichever matches the user's team). For total, note `line`.
- **If no speculationId for the desired market type** → you will use the contestId quote path in Step 2. Note the `contestId`.

If the team is not found → respond: "No active market found for {input}."

## Step 2: Get a Quote

Call `get-odds-history` with the contestId. Select odds for the detected market type:
- **Moneyline:** `current.moneyline.awayOdds` or `homeOdds`
- **Spread:** `current.spread.awayOdds` or `homeOdds`
- **Total:** `current.total.overOdds` or `underOdds`

Apply the odds multiplier and floor to 2 decimal places: `Math.floor(marketOdds * oddsMultiplier * 100) / 100`. Higher decimal odds = higher payout for the bettor.

If no odds data for the relevant market type: use 1.91 for spread/total. For moneyline, ask the user.

**Request a quote using `get-instant-match-quote`:**

- **If speculationId exists** (from Step 1): pass the `speculationId`.
- **If no speculationId**: pass the `contestId`, `marketType`, and `line` instead. The server resolves the speculation automatically.

Include: `side` ("home"/"away"/"over"/"under"), `amountUSDC`, `odds`, `oddsFormat: "decimal"`, `wallet`.

**For `side`:** Moneyline/spread: homeTeam → "home", awayTeam → "away". Total: "over" or "under".

**Save the `txParams` object from the response.** This contains all pre-computed on-chain parameters.

- If `approved` is false → tell the user "Michelle (market maker agent) declined — {reason}" and stop.
- If `approvedOddsDecimal` >= your requested odds → proceed silently.
- If `approvedOddsDecimal` < your requested odds → **stop and confirm with the user.** Show the counter-offer and expiry. If accepted, call `accept-counter-offer` with the quoteId and wallet. **Use the txParams from the accept-counter response.**

**Line reporting:** For spread, use `awayLine`/`homeLine` from the speculation (Step 1), not odds-history. For total, use `line` from the speculation. Use these in all user-facing messages.

## Step 3: Create Position On-Chain

Write and execute a **single Node.js script** (ethers.js v6) that does everything below in sequence.

**Transaction safety (mandatory):** Before signing any transaction, verify ALL of the following. If any check fails, STOP and report to the user.

1. **Contract address** equals the PositionModule address: `0xF717aa8fe4BEDcA345B027D065DA0E1a31465B1A`. Never derive a contract address from an API response.
2. **Method name:** `txParams.method` must be `"createUnmatchedPair"` OR `"createUnmatchedPairWithSpeculation"`.
3. **Amount** must not exceed the 3 USDC cap (plus any user-requested contribution).
4. **Private key handling:** OSPEX_WALLET_PRIVATE_KEY must only be used inside ethers.js `Wallet` construction. Never log, echo, interpolate into URLs, or pass to any external service.

```javascript
const VALID_METHODS = ["createUnmatchedPair", "createUnmatchedPairWithSpeculation"];
if (!VALID_METHODS.includes(txParams.method)) {
  throw new Error(`Unexpected txParams.method: ${txParams.method}`);
}
```

**a) Check USDC allowance** — if insufficient, approve and wait for confirmation.

**b) Create the position:** Call `positionModule[txParams.method]()`, passing `txParams.args` values directly. Do not compute odds, timestamps, positionType, or any parameter yourself.

- For `createUnmatchedPair`: `(speculationId, odds, unmatchedExpiry, positionType, amount, contributionAmount)`
- For `createUnmatchedPairWithSpeculation`: `(contestId, scorer, theNumber, leaderboardId, odds, unmatchedExpiry, positionType, amount, contributionAmount)`

Wait for mining: `const receipt = await tx.wait();`

**Critical — no retries after a successful transaction:** Once `tx.wait()` confirms, the position exists on-chain and funds have moved. If any subsequent step fails, DO NOT re-run the script. Report the tx hash and stop.

Contributions: `contributionAmount` defaults to "0". Only add one if the user explicitly requests it (e.g., "Lakers, tip 1 USDC") — scale to 6 decimals.

## Step 4: Match and Report

**Get positionId:** Call `get-position-by-tx` with the transaction hash.

**Match the position:** If you have an API key, call `execute-instant-match` with the quoteId and positionId. Otherwise, call the REST endpoint directly as a fallback:

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

**Odds formatting:** Prepend `+` to positive American odds (e.g., `110` → `+110`).

If `matchedAmountUSDC`/`potentialPayoutUSDC` absent from match response, use quoted amount and `amount x approvedOddsDecimal`.

**Report:**
```
Done. {Team} {ML/spread line/total line} at {americanOdds} ({decimalOdds}x), {matchedAmountUSDC} USDC matched, potential payout {potentialPayoutUSDC} USDC.
https://ospex.org/p/{positionId}
```

If matching fails after 5 retries, tell the user the position was created (share tx hash) but instant match couldn't complete. Michelle may still match it during normal processing. The user can withdraw anytime if it stays unmatched.

## Additional Operations

Only triggered when the user explicitly requests (e.g., "status", "withdraw", "claim").

### Status
Call `get-position-status` with the wallet address. Show the `formatted` text. If `claimable` is non-empty, offer to claim.

### Withdraw
Call `get-withdraw-params`. Execute a Node.js script calling `positionModule.adjustUnmatchedPair()` using `txParams.args` directly. Verify `txParams.method` equals `"adjustUnmatchedPair"`. Call `get-withdraw-result` with the tx hash. Report: `Withdrawn. {amountReturned} USDC returned. Tx: {txHash}`

### Claim
Call `get-claim-params`. Execute a Node.js script calling `positionModule.claimPosition()` for each position using `txParams.args`. Verify `txParams.method` equals `"claimPosition"`. Call `get-claim-result` with the tx hash. Report: `Claimed {payout} USDC. Tx: {txHash}`

| Situation | Function | When |
|-----------|----------|------|
| Unmatched, game hasn't started | `adjustUnmatchedPair` | Speculation open |
| Game ended, won or has remainder | `claimPosition` | Speculation settled |

## On-Chain Reference

Network: Polygon (chain ID 137).

| Contract | Address |
|----------|---------|
| USDC | `0x3c499c542cEF5E3811e1192ce70d8cC03d5c3359` (6 decimals) |
| PositionModule | `0xF717aa8fe4BEDcA345B027D065DA0E1a31465B1A` |

For contract parameter details (odds conversion, theNumber, bounds, error cases), load `{baseDir}/references/contract-interface-reference.md`.

**Minimal ABI (ethers.js v6 human-readable):**
```json
[
  "function approve(address spender, uint256 amount) returns (bool)",
  "function allowance(address owner, address spender) view returns (uint256)",
  "function createUnmatchedPair(uint256 speculationId, uint64 odds, uint32 unmatchedExpiry, uint8 positionType, uint256 amount, uint256 contributionAmount)",
  "function createUnmatchedPairWithSpeculation(uint256 contestId, address scorer, int32 theNumber, uint256 leaderboardId, uint64 odds, uint32 unmatchedExpiry, uint8 positionType, uint256 amount, uint256 contributionAmount)",
  "function adjustUnmatchedPair(uint256 speculationId, uint128 oddsPairId, uint32 newUnmatchedExpiry, uint8 positionType, int256 amount, uint256 contributionAmount)",
  "function claimPosition(uint256 speculationId, uint128 oddsPairId, uint8 positionType)"
]
```

## Gotchas

- **Firebase indexing delay:** 5-25s between on-chain tx confirmation and position appearing in the API. The match retry loop in Step 4 handles this.
- **Odds multiplier rationale:** Requesting 5% above market makes the bet more likely to be approved by Michelle while still getting competitive odds.
- **`data` vs `formatted`:** All API responses use a `{ data, formatted }` envelope. The `formatted` field is for display. Never use it for branching — always use `data`.
- **Counter-offer TTL:** Default 180s. The user must accept and Steps 3-4 must complete before `expiresAt`.
- **Two-path speculation logic:** If `speculations` has a matching speculationId → quote with speculationId, expect `txParams.method = "createUnmatchedPair"`. If no matching speculationId → quote with contestId, expect `txParams.method = "createUnmatchedPairWithSpeculation"`. Both are valid. The transaction safety check must accept both.
- **Line validation:** Spread and total lines always end in .5. If the data doesn't conform, something is wrong; stop and report.
- **For spread markets:** Use `awayLine`/`homeLine` from the speculation for the user's team. Odds-history is for odds values only.
- **No retries after confirmed tx:** Once `tx.wait()` confirms, funds moved. Never re-run the script. Report tx hash and recover manually.
