# Crow — Give Your AI Agent a Wallet

[Crow](https://crowpay.ai) lets your AI agent pay for APIs and services autonomously — within your rules. Set spending limits, fund a wallet, and let your agent handle the rest.

**Two payment rails:**
- **x402 (USDC on Base)** — On-chain payments via the [x402 protocol](https://www.x402.org/)
- **Credit card** — Card payments via Stripe

## Quick Start

1. Copy `crow.md` and give it to your agent
2. The agent calls `POST /setup` to create a wallet and API key
3. Visit the claim URL to set your spending rules
4. Fund the wallet with USDC or add a credit card
5. Your agent handles payments automatically

## How to Give It to Your Agent

### Claude Code / CLAUDE.md

Add `crow.md` to your project root, or paste it into your `CLAUDE.md`:

```bash
curl -o crow.md https://raw.githubusercontent.com/Fallomai/skills/main/crow/crow.md
```

### ChatGPT / Custom GPTs

Paste the contents of `crow.md` into your system prompt or custom instructions.

### Any HTTP-capable agent

Your agent just needs to be able to:
1. Read the instructions in `crow.md`
2. Make HTTP requests to `https://api.crowpay.ai`

No SDK required — it's all REST.

## What Happens

1. Agent encounters a paywall (HTTP 402) or needs to pay a merchant
2. Agent calls Crow's `/authorize` endpoint
3. Crow checks your spending rules
4. If within rules → payment is auto-approved and signed
5. If above threshold → you get a notification to approve or deny
6. Agent completes the payment

## Links

- **Dashboard**: [crowpay.ai/dashboard](https://crowpay.ai/dashboard)
- **x402 Protocol**: [x402.org](https://www.x402.org/)
- **Crow Homepage**: [crowpay.ai](https://crowpay.ai)

## License

MIT
