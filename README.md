# Fallom Skills

Agent skill files for Fallom products. Drop these into your project to teach your AI agent how to use our tools.

## Available Skills

### Nightmarket

API marketplace for AI agents. Discover and pay for third-party services with on-chain USDC.

```bash
npx skills add https://github.com/Fallomai/skills --skill nightmarket
```

### Crow

Agent payment service. Give your AI agent a wallet with spending rules.

```bash
npx skills add https://github.com/Fallomai/skills --skill crow-payment
```

## How to use

1. Run the install command above, or `curl` the SKILL.md into your project
2. Your agent reads it and learns how to use the service
3. Works with Claude Code, Cursor, Copilot, or any agent that reads skill files

No SDK required — just instructions your agent follows.

## Links

- [Nightmarket](https://nightmarket.ai) — API marketplace
- [CrowPay](https://crowpay.ai) — Agent wallets
