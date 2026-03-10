You are creating an OpenClaw AgentSkill named ospex-one.

**Goal:** produce a single SKILL.md file that exactly matches ospex-one **version 1.3.3** behavior/spec.

This is a "generate-your-own-skill" prompt for users who prefer not to install from a third-party registry. The output SKILL.md must be self-contained and executable as an OpenClaw skill procedure.

## Output requirements (strict)

- Output **only** the contents of `SKILL.md` (Markdown). No commentary.
- Include YAML frontmatter with at least:
  - `name: ospex-one`
  - `description`: mention one-word sports bet; examples; supported leagues NBA/NHL/NCAAB
  - `version: 1.3.3`
  - `homepage: https://ospex.org`
  - `allowed-tools`: include what’s required to run Node and shell commands
  - `metadata` suitable for ClawHub/OpenClaw that declares:
    - Supported OSes
    - Required binaries: `curl`, `node`
    - Required env vars: `OSPEX_WALLET_PRIVATE_KEY`, `OSPEX_WALLET_ADDRESS`, `OSPEX_RPC_URL`
    - Optional install hint for ethers (npm)
- The body must be a complete, agent-executable procedure with clear steps and exact API calls / behaviors.

## Hard constraints (do not deviate)

- Keep user-facing messages minimal and operational.
- No marketing tone.
- No emojis in the SKILL body (frontmatter metadata may include platform emoji if standard).
- Do not add extra files. Do not reference any local scripts. Everything must be described within SKILL.md.
- Use Ospex API + on-chain calls exactly as specified below.
- **Critical:** For on-chain calls, do **not** compute transaction arguments yourself. You must use the `txParams` object returned by the quote/counter-accept endpoints.

## Required SKILL.md content (spec)

### Trust + risk

- Include a blunt "Trust statement" that this skill executes real transactions on Polygon mainnet using the configured wallet.
- Cap position size at **3 USDC** per transaction by default.
- Mention unmatched funds can be withdrawn at any time.
- Link to the canonical risks doc:
  - https://github.com/ospex-org/ospex-contracts-v2/blob/main/docs/RISKS.md

### Input expectations

- This skill is designed for single-word input — a team name, city, or abbreviation.
- If the user’s message contains additional modifiers (spread/total language, amount language, etc.), proceed if intent is unambiguous; otherwise ask a single clarification question.

### Defaults section

Define defaults in a small table:

- Market: moneyline (but allow override to spread/total via input like `Lakers -6.5` or `over 220.5`)
- Amount: 3 USDC (maximum per transaction)
- Side: the named team
- Odds multiplier: 1.05
- Odds: market odds × multiplier

### Progress Updates

Instruct the agent to send short, single-line progress updates at each stage:

- After Step 1 resolves the team:
  - `Found: {awayTeam} vs {homeTeam}, {matchTime formatted as 'Mon D, YYYY, H:MM PM' in EST, e.g. 'Mar 3, 2026, 8:00 PM EST'}, {marketType}{line if applicable}`
- After Step 2 gets a quote:
  - `Quote: {approvedOddsAmerican} ({approvedOddsDecimal}), {amount} USDC`
- After Step 3 creates the position:
  - `Position created, waiting for match...`
- Step 4 delivers the final result.

If any step fails, the error message is enough (no extra chatter). If any API call returns an unexpected error: stop and report it; do not silently retry/work around failures.

### Step 1 — Resolve team and market type

On any input: your **first action** is always to call the API (do not ask the user anything first).

Detect market type from input:
- If input includes a point value (e.g., `Atlanta -6.5`, `Boise +1.5`, `Celtics -3`) → market type = **spread**, line = that signed number
- If input includes `over` or `under` with a number (e.g., `Lakers over 220.5`) → market type = **total**, line = that number
- Else → market type = **moneyline**

For spread/total:
- This skill uses the **current market line** from odds-history.
- If the user-specified line differs from the market’s current line: inform the user of the current line and ask if they want to proceed at the market line. Do not create a position at a non-market line.

Procedure:
1. Call `GET /markets`.
2. Search all responses for the input text in `homeTeam` and `awayTeam`.
   - Note: API responses use `{ data: ..., formatted: "..." }`. When searching, look inside `data`.
3. If found in exactly one game: capture `contestId`, `matchTime`, whether the selected team is home/away.
   - Check the `speculations` array for an existing `speculationId` matching the detected market type; if present, save it.
   - If missing: ok; Step 2 can quote via the contest-path quote endpoint which creates speculation on-the-fly.
4. If not found: respond exactly `No active market found for {input}`.

Only ask for clarification if the team is genuinely ambiguous across multiple games.

### Step 2 — Get a quote from the agent market maker (Michelle)

First determine requested odds:
- Call `GET /analytics/odds-history/{contestId}`.
- Select current odds for the detected market type:
  - Moneyline: `current.moneyline.awayOdds` or `current.moneyline.homeOdds` depending on user team
  - Spread: `current.spread.awayOdds` or `current.spread.homeOdds`
  - Total: `current.total.overOdds` or `current.total.underOdds`

Apply odds multiplier and floor to 2 decimals:
- `Math.floor(marketOdds * oddsMultiplier * 100) / 100`

Notes:
- Higher decimal odds = better payout for the bettor.
- If odds-history returns no data:
  - spread/total: use **1.91** default
  - moneyline: ask the user what odds they want (do not guess)

Line validation:
- Quote API requires .5 increments for spread/total. If the odds-history line is a whole number or does not end in .5: stop and tell the user:
  - `Spread/total line unavailable for this market right now.`
  - Do not attempt to convert.

Quote request:

If `speculationId` exists:
```
POST /instant-match/{speculationId}/quote?stream=false
{
  "side": "home" | "away" | "over" | "under",
  "amountUSDC": 3,
  "odds": {calculatedOddsDecimal},
  "oddsFormat": "decimal",
  "wallet": "{OSPEX_WALLET_ADDRESS}"
}
```

If no `speculationId` exists:
```
POST /instant-match/quote?stream=false
{
  "contestId": {contestId},
  "marketType": "moneyline" | "spread" | "total",
  "side": "home" | "away" | "over" | "under",
  "amountUSDC": 3,
  "odds": {calculatedOddsDecimal},
  "oddsFormat": "decimal",
  "wallet": "{OSPEX_WALLET_ADDRESS}",
  "line": {lineValueForSpreadOrTotalOnly}
}
```

Both return: `quoteId`, `approved`, `approvedOddsDecimal`, `approvedOddsAmerican`, `expiresAt`, optional `counterOffer`, and **`txParams`**.

Rules:
- **Save `txParams`** — it contains all pre-computed on-chain parameters. Do not compute these values yourself.
- If `approved` is false: tell user `Michelle (market maker agent) declined — {reason}` and stop.
- If `approvedOddsDecimal >= requestedOdds`: proceed automatically.
- If `approvedOddsDecimal < requestedOdds`: stop and ask user to confirm accepting worse odds; warn the quote expires at `expiresAt`.
  - If user accepts, call:
```
POST /instant-match/{quoteId}/accept-counter
Body: { "wallet": "{OSPEX_WALLET_ADDRESS}" }
```
  - This returns updated `txParams`. Use the updated one.

Side mapping:
- moneyline/spread: user team == `homeTeam` → `home`; else `away`
- total: user said `over` → `over`; `under` → `under`

### Step 3 — Create position and match

Write and execute a **single Node.js script** (ethers.js v6) that does everything in sequence. Do not split into multiple script runs.

Use `txParams` directly:
- `txParams.method` is `createUnmatchedPair` or `createUnmatchedPairWithSpeculation`.
- Call the corresponding PositionModule function and pass `txParams.args.*` directly.
- Do not compute odds/timestamps/positionType/etc.

Contributions:
- `txParams.args.contributionAmount` defaults to `"0"`.
- Only change it if the user explicitly asks to tip/contribute.

After sending the transaction:
- Wait for confirmation: `const receipt = await tx.wait()`.

PositionId:
- Do **not** parse logs yourself. Call server endpoint:
  - `GET /positions/by-tx/{receipt.hash}`
- Use `positions[0].positionId`.

Match call:
- Wait 5 seconds for Firebase indexing.
- Retry up to 5 times with 5-second backoff only when response code is `POSITION_NOT_FOUND`.
- Call:
  - `POST /instant-match/{quoteId}/match` with JSON body `{ "positionId": "..." }`.

If matching fails after retries:
- Tell the user the position was created on-chain (share tx hash) but instant match could not be completed.
- Explain: position is live on the order book; may still match; if unmatched they can withdraw anytime.

### Step 4 — Report result

Odds formatting:
- If `approvedOddsAmerican` is a positive number without a leading `+`, prepend one (e.g., `110` → `+110`). Negative odds already include the `-`.

Match enrichment:
- The match endpoint may return `matchedAmountUSDC` and `potentialPayoutUSDC`. Use these if present.
- If absent, use the quoted amount and compute payout as `amount × approvedOddsDecimal`.

Final output format must be:
```
Done. {Team} {marketType abbreviation: ML/spread line/total line} at {americanOdds} ({decimalOdds}x), {matchedAmountUSDC} USDC matched, potential payout {potentialPayoutUSDC} USDC.
https://ospex.org/p/{positionId}
```

### Step 5 — Additional operations (only if user asks)

Include:

- Status check:
  - `GET /positions/{OSPEX_WALLET_ADDRESS}/status`
  - Show `formatted` response.
  - If `claimable` non-empty: offer to claim and, if confirmed, follow claim flow.

- Withdraw unmatched position:
  - `GET /positions/{OSPEX_WALLET_ADDRESS}/withdraw-params`
  - Use returned `txParams` and call `positionModule.adjustUnmatchedPair()` with args directly.
  - Then call `GET /positions/withdraw-result/{txHash}` and report amount returned.

- Claim resolved position:
  - `GET /positions/{OSPEX_WALLET_ADDRESS}/claim-params`
  - For each entry, call `positionModule.claimPosition()` with args directly.
  - Then call `GET /positions/claim-result/{txHash}` and report payout.

### API reference section

Include base URL:
- `https://api.ospex.org/v1` (no auth; rate limit 100 req/60s/IP)

List endpoints used:
- `GET /markets?sport={nba|nhl|ncaab}` (mention sport param usage)
- `GET /analytics/odds-history/{contestId}`
- `POST /instant-match/{speculationId}/quote?stream=false`
- `POST /instant-match/quote?stream=false`
- `POST /instant-match/{quoteId}/accept-counter`
- `POST /instant-match/{quoteId}/match`
- `GET /positions/by-tx/{txHash}`
- `GET /positions/{address}/status`
- `GET /positions/{address}/withdraw-params`
- `GET /positions/withdraw-result/{txHash}`
- `GET /positions/{address}/claim-params`
- `GET /positions/claim-result/{txHash}`

### On-chain reference section

Include:
- Network: Polygon (chain ID 137)
- Contract addresses:
  - USDC: `0x3c499c542cEF5E3811e1192ce70d8cC03d5c3359` (6 decimals)
  - PositionModule: `0xF717aa8fe4BEDcA345B027D065DA0E1a31465B1A`
- Minimal ABI (ethers.js v6 human-readable) including:
  - `approve`, `allowance`
  - `createUnmatchedPair`, `createUnmatchedPairWithSpeculation`
  - `adjustUnmatchedPair`, `claimPosition`
  - PositionCreated/PositionClaimed events

Also mention additional docs available:
- `{baseDir}/references/api-reference.md`
- `{baseDir}/references/contract-interface-reference.md`

## Final instruction

Now output the complete `SKILL.md` content for ospex-one **version 1.3.3**, following this spec exactly.