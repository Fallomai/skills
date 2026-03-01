---
name: nightmarket
description: Discover and call paid API services on the Nightmarket marketplace. Use when the user wants to find, browse, or call third-party APIs with automatic on-chain USDC payment via x402.
---

# Nightmarket — API Marketplace for AI Agents

Nightmarket lets agents discover and pay for third-party API services. Every call settles on-chain in USDC via the x402 protocol. No API keys, no subscriptions.

**Marketplace:** https://nightmarket.ai/marketplace

## When to Use This Skill

- User asks to find or discover an API service
- User wants to call a paid API through Nightmarket
- User needs to set up Nightmarket MCP or connect their agent
- User gets a 402 from a Nightmarket proxy URL

## Setup — Choose One

### Option A: MCP Server (recommended for Claude Code)

Add to your MCP configuration:

```json
{
  "mcpServers": {
    "nightmarket": {
      "command": "npx",
      "args": ["-y", "@nightmarket/mcp"],
      "env": {
        "WALLET_KEY": "<your-wallet-private-key>"
      }
    }
  }
}
```

This gives you three tools:
- `browse_services` — Search and list available APIs
- `get_service_details` — Full details, pricing, request/response examples
- `call_service` — Call an API with automatic USDC payment

### Option B: REST API (works with any agent)

No MCP required. Call services directly through the proxy:

```
GET/POST https://nightmarket.ai/api/x402/<endpoint_id>
```

Returns `402 Payment Required` with payment details on first call. See `references/rest-api.md`.

## Payment

### With CrowPay (recommended — no private key needed)

If Crow is set up, 402 payments are handled automatically. No `WALLET_KEY` needed.

Install the Crow skill:
```
npx skills add Fallomai/skills --skill crow
```

See `references/crow-payments.md` for the full integration flow.

### With a direct wallet

Set `WALLET_KEY` in your MCP config (Option A). The MCP server signs payments using your private key.

Need a wallet? Get one at https://crowpay.ai

## Quick Reference

| Action | MCP Tool | REST Endpoint |
|--------|----------|---------------|
| Browse services | `browse_services(search?)` | https://nightmarket.ai/marketplace |
| Service details | `get_service_details(endpoint_id)` | — |
| Call a service | `call_service(endpoint_id, method, body?)` | `POST /api/x402/<endpoint_id>` |

## References

- `references/rest-api.md` — Direct REST proxy usage and x402 flow
- `references/crow-payments.md` — CrowPay integration for automatic 402 handling
- `references/mcp-tools.md` — Detailed MCP tool parameters and responses
