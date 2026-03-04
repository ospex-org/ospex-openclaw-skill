# Ospex One-Word Betting — OpenClaw Setup Prompt

Paste the following into your OpenClaw chat to set up the skill.

---

## Setup Prompt

```
Create a skill called "ospex-one" for placing sports bets with minimal input on the Ospex protocol. Here's what I need:

**What the skill does:** I say a team name (like "Edmonton" or "Duke" or "fade the Celtics") and the agent places a position on the Ospex peer-to-peer sports prediction protocol on Polygon. One message in, one transaction hash out. No confirmation prompts, no intermediate questions.

**Skill description (use this exactly — it controls when the skill activates):**
"Bet on sports with one word. Say a team name, city, or abbreviation to place a position instantly. 'Edmonton', 'Duke spread', 'fade the Celtics', 'over on the Lakers game'. NBA, NHL, NCAAB. Zero-vig peer-to-peer protocol on Polygon."

**Defaults:**
- Market type: moneyline (override with "Duke spread" or "over Celtics")
- Amount: 3 USDC (override with "2 on Edmonton")
- Side: the named team (override with "fade Duke" for the opponent)
- No confirmation prompts by default

**The API:**
- Base URL: https://api.ospex.org/v1
- No auth required. Rate limit: 100/60s.
- GET /markets?sport={nba|nhl|ncaab} — find today's games, search homeTeam/awayTeam for the resolved team
- GET /odds?sport={sport} — check existing orderbook liquidity
- GET /markets/{contestId} — get orderbook depth for a specific game
- POST /instant-match/{speculationId}/quote?stream=false — request a quote from the AI market maker (body: side, amount, odds, oddsFormat, wallet, feeTxHash)
- POST /instant-match/{quoteId}/match — execute the match (body: positionId)
- All responses return { "data": ..., "formatted": "..." }

**On-chain (Polygon mainnet, chain ID 137):**
- USDC: 0x3c499c542cEF5E3811e1192ce70d8cC03d5c3359 (6 decimals)
- PositionModule: 0xF717aa8fe4BEDcA345B027D065DA0E1a31465B1A
- Fee Wallet: 0xdaC630aE52b868FF0A180458eFb9ac88e7425114
- To create a position: approve USDC on the USDC contract, then call create on PositionModule
- positionType 0 = upper (away wins / away covers / over)
- positionType 1 = lower (home wins / home covers / under)

**Flow when I say a team name:**
1. Resolve to the full team name and sport
2. GET /markets to find today's game
3. GET /odds to check for existing orderbook liquidity
4. If good odds exist, take them. If not, POST /instant-match quote
5. Execute on-chain: approve USDC, create position, trigger match
6. Report: team, opponent, market type, odds, amount, tx hash

**Team resolution:** Map city names, abbreviations, and mascots to full team names. "edmonton"/"oilers"/"edm" = Edmonton Oilers (nhl). If ambiguous, check which sport has a game today. If no game today, say so and check the schedule.

**If I say "status":** show wallet balance and open positions.
**If I say "history":** show recent positions and outcomes.

Make the skill. Put it in my workspace skills directory. Use the emoji 🎯 in the metadata.
```

---

## After Setup

Once the skill is created, restart your OpenClaw gateway and verify with:

```
openclaw skills list
```

You should see `ospex-one` in the list. Test with:
- "What NBA games are on today?" — should return game list
- "Edmonton" — should resolve to Oilers and begin the positioning flow

## Requirements

- A Polygon wallet with USDC and POL for gas
- `curl` available on your system (most systems have this already)
- OpenClaw with `exec` tool enabled

## Other Ospex Skills

- **ospex-research** — Analytics, ELO ratings, matchup analysis, injury reports, expert picks. For doing homework before you bet.
- **ospex-markets** — Browse available markets, check odds, view leaderboard standings. For seeing what's out there.

Setup prompts for these skills: https://ospex.org/agents

## Learn More

- Protocol: https://ospex.org
- API docs: https://api.ospex.org/v1/protocol/info
- Risk assessment: https://ospex.org/risks
- Source: https://github.com/nickthorpe71/ospex-agent-server

## Disclaimer

Ospex is an unaudited peer-to-peer sports prediction protocol. Positions are limited to 3 USDC. Users are responsible for compliance with local laws. This is not financial advice.
