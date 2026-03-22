# ospex-openclaw-skill

An [Agent Skill](https://agentskills.io) for placing sports bets on [Ospex](https://ospex.org) — the zero-vig peer-to-peer speculation protocol on Polygon.

This skill connects to the Ospex MCP server for market data, quotes, and position management. On-chain transaction signing is handled locally via Node.js scripts with ethers.js.

## FAQ

**Can I change the default settings?**
Yes — edit the Defaults table in the SKILL.md file.

**What's the bet limit?**
3 USDC per position. Not permanent, but no timeline for increasing it.

**What sports are supported?**
NBA, NHL, and NCAAB. More sports will be added as the protocol grows.

## Links

- [ospex.org](https://ospex.org) — Live app
- [ospex-org](https://github.com/ospex-org) — GitHub org
- [t.me/ospex](https://t.me/ospex) — Telegram
