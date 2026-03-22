# ospex-openclaw-skill

An [Agent Skill](https://agentskills.io) for placing sports bets on [Ospex](https://ospex.org) — the zero-vig peer-to-peer speculation protocol on Polygon.

This skill connects to the Ospex MCP server for market data, quotes, and position management. On-chain transaction signing is handled locally via Node.js scripts with ethers.js. NBA, NHL, NCAAB.

## How It Works

Say a team name — "Lakers", "Celtics", "Duke" — and the agent finds the game, gets a quote from the market maker, creates a position on-chain, and matches it. Supports moneyline (default), spread ("Lakers -6.5"), and total ("over 220.5") markets.

All positions are capped at 3 USDC. Unmatched positions can be withdrawn at any time.

## Setup

1. **Use a dedicated wallet.** Do not use your primary wallet. Create a new one, fund it with a small amount of USDC on Polygon, and use that for this skill.
2. **Secure your environment file.** Your private key should be in a file with restricted permissions: `chmod 600` on your `.env` or gateway config file. Never commit it to version control.
3. **Fund conservatively.** Keep only what you're willing to lose in the wallet — enough for a few bets plus gas. On Polygon, gas costs are negligible (fractions of a cent per transaction).

**Required environment variables:**
- `OSPEX_WALLET_PRIVATE_KEY` — Dedicated wallet private key (high-sensitivity)
- `OSPEX_WALLET_ADDRESS` — Wallet address
- `OSPEX_RPC_URL` — Polygon RPC endpoint

## FAQ

**Can I change the default settings?**
Yes — edit the Defaults table in the SKILL.md file.

**What's the bet limit?**
3 USDC per position. Not permanent, but no timeline for increasing it.

**What sports are supported?**
NBA, NHL, and NCAAB. More sports will be added as the protocol grows.

**What if a market doesn't exist yet for my game?**
The skill handles this automatically. If no speculation exists for the desired market type, it creates both the speculation and the position atomically in a single transaction.

## Publishing

```bash
claw publish ospex/
```

## Links

- [ospex.org](https://ospex.org) — Live app
- [ospex-org](https://github.com/ospex-org) — GitHub org
- [t.me/ospex](https://t.me/ospex) — Telegram
