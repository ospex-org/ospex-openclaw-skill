You are creating an OpenClaw AgentSkill named ospex-one.

**Goal:** produce a single SKILL.md file that implements "bet on sports with one word" behavior: the user types a single team name / city / abbreviation (e.g. "Edmonton", "Duke", "Celtics", "Lakers") and the agent executes the full flow to place a small, capped bet (default 3 USDC) on Polygon via Ospex, returning a transaction hash + position link. The skill should emphasize execution over discussion and keep user-facing messages minimal.

## Output requirements

- Output only the contents of SKILL.md (Markdown).
- Include required YAML frontmatter with at least:
  - `name`: ospex-one
  - `description`: mention one-word sports bet; examples; supported leagues NBA/NHL/NCAAB
  - A version number
  - `homepage`: `https://ospex.org`
  - `allowed-tools` appropriate for running scripts/commands (match the platform conventions)
  - A `metadata` section suitable for ClawHub/OpenClaw that declares:
    - Supported OSes
    - Required binaries (`curl`, `node`)
    - Required env vars: `OSPEX_WALLET_PRIVATE_KEY`, `OSPEX_WALLET_ADDRESS`, `OSPEX_RPC_URL`
    - Optional install hint for ethers (npm)
- The body must be a complete, agent-executable procedure with clear steps and exact API calls / behaviors.

## Behavioral spec

Must be encoded as the SKILL.md procedure.

### Trust + risk

- Include a blunt "Trust statement" that this skill executes real transactions on Polygon mainnet using the configured wallet.
- Cap position size at 3 USDC per transaction by default.
- Mention unmatched funds can be withdrawn.
- Link to the canonical Ospex risks doc (available in references).

### Defaults section

Define defaults in a small table:

- Market: moneyline
- Amount: 3 USDC (max)
- Side: the named team
- Odds multiplier: 1.05
- Odds: market odds × multiplier (floor to 2 decimals)

### Position type mapping

Include a compact table mapping `positionType` for Moneyline/Spread/Total:

- `0` = Away / Over
- `1` = Home / Under

Note: moneyline is the default market, but the skill may be configured for others.

### Progress updates

Instruct the agent to send short, single-line progress updates at key points, using these exact templates:

- After resolving the team: `Found: {awayTeam} vs {homeTeam}, {matchTime formatted as 'Mon D, YYYY, H:MM PM' in local time, e.g. 'Mar 3, 2026, 8:00 PM CST'}`
- After receiving a quote: `Quote: {approvedOddsAmerican} ({approvedOddsDecimal}), {amount} USDC`
- After creating the position: `Position created, waiting for match...`
- Final completion line includes a position URL (see Step 4)

If a step fails, the error message is enough (no extra chatter).

### Step 1 — Resolve team from markets

- On any input, the first action is API lookup (no questions first).
- Call:
  - `GET /markets?sport=nba`
  - `GET /markets?sport=nhl`
  - `GET /markets?sport=ncaab`
- Search `homeTeam` and `awayTeam` for the input text.
- If found in exactly one game: capture `contestId`, `matchTime`, whether selected team is home/away, and whether `speculations` already contains a `speculationId` for moneyline.
- If not found: respond exactly `No active market found for {input}`.
- Only ask for clarification if ambiguous across multiple games.

### Step 2 — Get a quote from the agent market maker ("Michelle")

- Determine requested odds by calling `GET /analytics/odds-history/{contestId}` and using the appropriate current moneyline odds (home or away) in decimal.
- Apply odds multiplier; floor to 2 decimals using: `Math.floor(marketOdds * oddsMultiplier * 100) / 100`.
- If odds-history has no data: ask the user what odds they want (do not guess).
- If a relevant `speculationId` exists, request quote via:
  - `POST /instant-match/{speculationId}/quote?stream=false`
- Else request quote via:
  - `POST /instant-match/quote?stream=false`
- Both quote payloads must include: `side` (home/away), `amountUSDC`, `odds`, `oddsFormat` = `"decimal"`, `wallet`.
- Parse response fields: `quoteId`, `approved`, `approvedOddsDecimal`, `approvedOddsAmerican`, `expiresAt`, optional `counterOffer`.
- If `approved` is false: stop and tell the user Michelle declined, including reason.
- If approved odds are >= requested: proceed automatically.
- If approved odds are < requested: stop and ask user to confirm accepting worse odds; warn they have ~1 minute; if yes call:
  - `POST /instant-match/{quoteId}/accept-counter` with body `{ "wallet": "{OSPEX_WALLET_ADDRESS}" }`
- Use `approvedOddsDecimal` for the on-chain transaction.
- The match endpoint accepts an optional `feeTxHash` field, but fees are currently disabled — omit the field entirely.
- Side mapping: user team equals `homeTeam` → `"home"`, else `"away"`.

### Step 3 — Create position and match (Node.js + ethers v6)

Strongly recommend doing Step 3 as a single Node.js script using ethers.js v6 and `fetch()` for the match call, to avoid curl JSON escaping issues that have caused match failures in testing.

The script must:

1. Check USDC allowance; if insufficient, approve and wait for confirmation.
2. Create the unmatched position using the correct PositionModule function:
   - `createUnmatchedPair` if `speculationId` exists
   - `createUnmatchedPairWithSpeculation` if not (with moneyline scorer address)
   - Odds conversion: `Math.round(approvedOddsDecimal * 10000000)` (1e7 precision)
   - Set `unmatchedExpiry` to contest start time (unix seconds)
   - Amount in USDC 6 decimals
   - `contributionAmount` defaults to `0`; do not contribute unless user explicitly asks
3. Parse the `PositionCreated` event to extract `oddsPairId`.
   - Include a warning about ethers v6 `parseLog()` returning null for non-matching logs (must check for null).
4. Construct `positionId` as `{speculationId}_{wallet lowercase}_{oddsPairId}_{positionType}`.
5. Wait for Firebase indexing and call the match endpoint with retries:
   - Initial 5-second wait, up to 5 attempts with 5-second delays on `POSITION_NOT_FOUND`
   - `POST /instant-match/{quoteId}/match` with body `{ "positionId": "..." }`

If matching still fails after retries: tell user the position was created on-chain (share tx hash) but instant match failed; the position is live on the order book and may still match; if unmatched, can withdraw anytime.

### Step 4 — Report result

Final output line format must be exactly:

```
Done. {Team} ML at {americanOdds} ({decimalOdds}), {amount} USDC.
https://ospex.org/p/{positionId}
```

### Step 5 — Additional operations (only if user asks)

Add three subsections:

- **Status:** When user says "status", "how am I doing", or similar — call `GET /v1/positions/{OSPEX_WALLET_ADDRESS}/status` and show the `formatted` text from the response. If the `claimable` array is non-empty, offer to claim: `"You have {N} position(s) ready to claim (~{total} USDC). Want me to claim them?"` If confirmed, claim each using the `speculationId`, `oddsPairId`, and `positionType` from the response.
- **Withdraw unmatched position:** Use `adjustUnmatchedPair` with negative amount (6 decimals). Precondition: speculation still open. Report: `Withdrawn. {amount} USDC returned. Tx: {txHash}`
- **Claim resolved position:** Use `claimPosition`, parse `PositionClaimed` event to estimate payout if possible. Report: `Claimed {claimAmount} USDC. Tx: {txHash}`

### API reference section

Include base URL `https://api.ospex.org/v1`, note no auth + rate limit, and list the endpoints used in the workflow (including the status endpoint).

### On-chain reference section

- Network: Polygon (chainId `137`)
- List contract addresses for USDC, PositionModule, and moneyline scorer (from references / canonical docs).
- Provide a minimal human-readable ABI list suitable for ethers v6.
- Note that additional reference docs are available for deeper parameter details, error cases, and full API documentation:
  - `{baseDir}/references/api-reference.md`
  - `{baseDir}/references/contract-interface-reference.md`

## Style constraints

- Keep it decisive and procedural.
- No marketing tone.
- No emojis in the SKILL body (frontmatter metadata may include platform emoji if standard).
- Prefer exact strings and code blocks the agent can copy/paste and run.
- Don't add extra files; SKILL.md may reference the docs in the `references/` directory for deeper parameter details.

Now produce the complete SKILL.md content.