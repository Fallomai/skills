# Fallom Skills

Agent skill files for Fallom products. Drop these into your project to teach your AI agent how to use our tools.

## Available Skills

### Nightmarket

API marketplace for AI agents. Discover and pay for third-party services with on-chain USDC.

```bash
curl -o nightmarket.md https://raw.githubusercontent.com/Fallomai/skills/main/nightmarket/nightmarket.md
```

### Crow

Agent payment service. Give your AI agent a wallet with spending rules.

```bash
curl -o crow.md https://raw.githubusercontent.com/Fallomai/skills/main/crow/crow.md
```

## How to use

1. `curl` the skill file into your project root
2. Your agent reads it and learns how to use the service
3. Works with Claude Code, ChatGPT, or any agent that can read files and make HTTP requests

No SDK required — just instructions your agent follows.

## Links

- [Nightmarket](https://nightmarket.ai) — API marketplace
- [CrowPay](https://crowpay.ai) — Agent wallets
