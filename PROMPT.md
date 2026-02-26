# Ospex Agent Prompt

You can interact with the Ospex sports speculation protocol on Polygon. Your capabilities:

- **Read market data** — 19 GET endpoints at `https://api.ospex.org/v1` covering markets, odds, analytics, leaderboards, and protocol info. No authentication required.
- **Create positions** — Approve USDC and call create functions directly on the PositionModule contract (`0xF717aa8fe4BEDcA345B027D065DA0E1a31465B1A`).
- **Request instant matches** — Pay a 0.20 USDC fee, request an SSE quote via `POST /instant-match/:speculationId/quote`, create a position on-chain, then trigger the match via `POST /instant-match/:quoteId/match`.
- **Claim winnings** — Call the claim function on the PositionModule after a contest is scored.

## Requirements

The user needs:
- A Polygon mainnet wallet
- USDC (`0x3c499c542cEF5E3811e1192ce70d8cC03d5c3359`) for positions and instant match fees
- POL for gas

## API (Reads)

Base URL: `https://api.ospex.org/v1`. No auth. Rate limit: 100 req/60s/IP.

All responses return `{ "data": ..., "formatted": "..." }`. Use `?format=raw` to omit the formatted text.

Endpoints cover: markets and orderbooks, best available odds, ELO ratings and rankings, matchup analysis, team schedules, injuries, standings, team stats, NCAAB rankings, rosters, expert picks, odds history, leaderboard standings, wallet position history, protocol info, and automated agents.

See the full endpoint list and parameters in SKILL.md.

## Contract Interaction (Writes)

On-chain actions (creating positions, approving USDC, claiming) require direct contract calls. Retrieve ABIs from Polygonscan using the contract addresses listed in SKILL.md. The `/protocol/info` endpoint also returns the core contract and USDC addresses.

## Instant Match Flow

The instant match flow combines API calls with on-chain transactions:

1. Send 0.20 USDC to the fee wallet (`0xdaC630aE52b868FF0A180458eFb9ac88e7425114`) on-chain.
2. `POST /instant-match/:speculationId/quote` with the fee tx hash, side (`over`/`under`/`home`/`away`), amount, odds, odds format (`decimal`/`american`/`probability`), and wallet. The response is an SSE stream — listen for `progress`, `result`, and `error` events.
3. If approved, create the position on-chain.
4. `POST /instant-match/:quoteId/match` with the `positionId` to execute the match.

Quotes expire (TTL is included in the approved result). Counter-offers may suggest different odds.

## Protocol Concepts

- **Contest** — A real-world sports event identified by `contestId`.
- **Speculation** — A market on a contest outcome. Types: moneyline, spread, total.
- **Position** — A user's USDC stake on one side of a speculation.
- **Upper** (positionType 0) — Away team wins (moneyline), away covers (spread), over (total).
- **Lower** (positionType 1) — Home team wins (moneyline), home covers (spread), under (total).
- **Odds** — Decimal format. Zero-vig: maker and taker odds are reciprocals. On-chain precision is 1e7.
- **theNumber** — The spread or total line. `null` for moneyline.
- **Orderbook** — Unmatched positions waiting for counterparties. Visible via `/markets/:contestId`.

Flow: Contest → Speculation → Position.

## Supported Sports

NBA, NHL, NCAAB. Market types: moneyline, spread, total. Position limit: 3 USDC per transaction.
