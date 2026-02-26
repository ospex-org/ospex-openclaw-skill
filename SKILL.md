---
name: ospex
description: "Read sports markets, create positions, request instant matches, and claim winnings on the Ospex protocol (Polygon). Zero-vig peer-to-peer orderbook. Public API for reads, direct contract interaction for writes."
version: 2.0.0
homepage: "https://ospex.org"
metadata: {"openclaw":{"emoji":"⚖️","os":["darwin","linux","win32"]}}
---

# Ospex

Ospex is a zero-vig peer-to-peer sports speculation protocol on Polygon. Users take positions against each other through an on-chain orderbook — no house edge, no spread applied by the protocol. Positions are denominated in USDC. The protocol is permissionless: anyone with a Polygon wallet, USDC, and POL (for gas) can participate. All contracts are verified on Polygonscan.

Ospex is unaudited software. Before interacting with the protocol, read the [risk assessment](references/risks.md).

## Chain & Contract Info

| Item | Value |
|------|-------|
| Network | Polygon (chain ID 137) |
| RPC | Any Polygon mainnet RPC (`https://polygon-rpc.com`, Alchemy, Infura, etc.) |
| USDC | `0x3c499c542cEF5E3811e1192ce70d8cC03d5c3359` (native USDC, 6 decimals) |
| PositionModule | `0xF717aa8fe4BEDcA345B027D065DA0E1a31465B1A` |
| Fee Wallet | `0xdaC630aE52b868FF0A180458eFb9ac88e7425114` |

All contract addresses (13 modules) can be discovered via the `/protocol/info` API endpoint. ABIs are available on Polygonscan by searching the contract address.

**Full contract address table:**

| Contract | Address |
|----------|---------|
| OspexCore | `0x8016b2C5f161e84940E25Bb99479aAca19D982aD` |
| PositionModule | `0xF717aa8fe4BEDcA345B027D065DA0E1a31465B1A` |
| SpeculationModule | `0x599FFd7A5A00525DD54BD247f136f99aF6108513` |
| ContestModule | `0x9E56311029F8CC5e2708C4951011697b9Bb40A09` |
| OracleModule | `0x5105b835365dB92e493B430635e374E16f3C8249` |
| LeaderboardModule | `0xEA6FF671Bc70e1926af9915aEF9D38AD2548066b` |
| RulesModule | `0xEfDf69ef9f3657d6571bb9c979D2Ce3D7Afb6891` |
| TreasuryModule | `0x48Fe67B7b866Ce87eA4B6f45BF7Bcc3cf868ccD0` |
| SecondaryMarketModule | `0x85E25F3BC29fAD936824ED44624f1A6200F3816E` |
| ContributionModule | `0x384e356422E530c1AAF934CA48c178B19CA5C4F8` |
| MoneylineScorerModule | `0x82c93AAf547fC809646A7bEd5D8A9D4B72Db3045` |
| SpreadScorerModule | `0x4377A09760b3587dAf1717F094bf7bd455daD4af` |
| TotalScorerModule | `0xD7b35DE1bbFD03625a17F38472d3FBa7b77cBeCf` |
| USDC | `0x3c499c542cEF5E3811e1192ce70d8cC03d5c3359` |

## Reading Market Data (API)

**Base URL:** `https://api.ospex.org/v1`

No authentication required. Rate limit: 100 requests per 60 seconds per IP.

All responses return `{ "data": ..., "formatted": "..." }`. Add `?format=raw` to omit the `formatted` field.

### Markets (3 endpoints)

| Endpoint | Description |
|----------|-------------|
| `GET /markets` | List upcoming markets. Filter by `sport`, `status`, `window` (hours). |
| `GET /markets/:contestId` | Single market with orderbook depth (unmatched positions per speculation). |
| `GET /odds` | Best available taker odds and liquidity per side (summary view). |

### Analytics (12 endpoints)

| Endpoint | Description |
|----------|-------------|
| `GET /analytics/elo/:teamName` | Single team ELO rating, record, trend, top-50 record. |
| `GET /analytics/elo/rankings` | Top teams by ELO. Filter by `sport`, `top`. |
| `GET /analytics/elo/all` | All ELO ratings for a sport. |
| `GET /analytics/matchup/:contestId` | Comprehensive matchup analysis (ELO prediction, schedule, injuries, standings, rankings, expert picks). |
| `GET /analytics/schedule/:teamName` | Recent/upcoming games, rest days, back-to-back status. |
| `GET /analytics/injuries/:teamName` | Current injury report. NBA and NHL only. |
| `GET /analytics/standings` | League standings by conference. NBA and NHL only. |
| `GET /analytics/team-stats/:teamName` | PPG, shooting percentages, special teams. NBA and NHL only. |
| `GET /analytics/rankings` | AP Top 25 or Coaches Poll. NCAAB only. |
| `GET /analytics/rosters/:teamName` | Current roster. NBA and NHL only. |
| `GET /analytics/expert-picks/:contestId` | Attributed pundit picks with reasoning. |
| `GET /analytics/odds-history/:contestId` | Opening lines, current lines, and line movement. |

### Leaderboard (2 endpoints)

| Endpoint | Description |
|----------|-------------|
| `GET /leaderboard` | Active leaderboard standings ranked by declared bankroll. |
| `GET /leaderboard/:address` | Position history for a wallet address with pagination. |

### Protocol (2 endpoints)

| Endpoint | Description |
|----------|-------------|
| `GET /protocol/info` | Network, contract addresses, supported sports, fee structure, leaderboard status. |
| `GET /protocol/agents` | Known automated agents with descriptions and last run times. |

For full endpoint documentation (parameters, response shapes, error codes), see [references/api-reference.md](references/api-reference.md).

## Creating Positions (Direct Contract Interaction)

Creating a position is a two-step on-chain process:

1. **Approve USDC** — Call `approve()` on the USDC contract (`0x3c499c542cEF5E3811e1192ce70d8cC03d5c3359`) to allow the PositionModule to spend your USDC.
2. **Create position** — Call the appropriate create function on the PositionModule (`0xF717aa8fe4BEDcA345B027D065DA0E1a31465B1A`).

Read the verified contract ABI on Polygonscan for function signatures and parameter details.

Creating a position does not guarantee a match. Your position may sit in the orderbook as an unmatched position until a counterparty takes the other side.

## Instant Match (API + On-Chain)

Instant match provides guaranteed liquidity from the protocol's automated market maker. It costs a 0.20 USDC fee per quote request.

**Four-phase flow:**

1. **Pay fee** — Send 0.20 USDC to the fee wallet (`0xdaC630aE52b868FF0A180458eFb9ac88e7425114`) on-chain. Save the transaction hash.
2. **Request quote** — `POST /instant-match/:speculationId/quote` with the fee transaction hash, desired side, amount, odds, and wallet. This returns a Server-Sent Events (SSE) stream with progress updates and a final result (approved, rejected, or error). Approved results include a `quoteId` and may include a `counterOffer` with alternative odds.
3. **Create position** — If approved, create your position on-chain (see Creating Positions above). Save the `positionId`.
4. **Trigger match** — `POST /instant-match/:quoteId/match` with your `positionId`. The market maker executes the match on-chain and returns a transaction hash.

For request/response shapes and error codes, see [references/api-reference.md](references/api-reference.md).

## Claiming Settled Positions

After a contest completes and is scored, winning positions can be claimed. Call the claim function on the PositionModule (`0xF717aa8fe4BEDcA345B027D065DA0E1a31465B1A`). Read the verified contract ABI on Polygonscan for the function signature.

Settlement is automatic — an off-chain scorer service verifies results and submits them on-chain via Chainlink Functions every 15 minutes.

## Supported Sports & Markets

**Sports:** NBA, NHL, NCAAB. NCAAB has the deepest analytical tooling (ELO, rankings, matchup analysis). NBA and NHL are supported but with sparser data coverage (no rankings endpoint; injuries, standings, and rosters are available).

**Market types:**

| Type | Description |
|------|-------------|
| Moneyline | Which team wins |
| Spread | Margin of victory (e.g., -3.5) |
| Total | Combined score over/under (e.g., 220.5) |

**Upper and Lower sides:**

| Side | Moneyline | Spread | Total |
|------|-----------|--------|-------|
| **Upper** (positionType 0) | Away team wins | Away team covers | Over |
| **Lower** (positionType 1) | Home team wins | Home team covers | Under |

## Key Facts

- **Zero vig.** Maker and taker odds are reciprocals. No spread or margin applied by the protocol.
- **USDC denominated.** All amounts are in USDC (6 decimals on-chain, human-readable in API responses).
- **Low gas.** Polygon L1 — transactions cost fractions of a cent in POL.
- **Leaderboard.** Weekly competitions with USDC prize pools. Players register bankrolls and compete for ROI rankings.
- **On-chain odds precision.** 1e7 (e.g., 1.80 odds = 18,000,000 on-chain). The API returns converted decimal values.
- **Per-transaction limit.** 3 USDC per transaction during the current phase.

## Risks

Key risks to understand before using Ospex:

1. **No professional audit.** Smart contracts have not undergone a formal security audit.
2. **Centralized admin key.** The deployer wallet can swap modules, redirect fees, and sweep unclaimed prizes.
3. **Thin liquidity.** The orderbook depends primarily on a single automated market maker.

For the complete risk assessment, see [references/risks.md](references/risks.md).

## Jurisdiction

Ospex does not restrict access by jurisdiction. Users are responsible for compliance with local laws.
