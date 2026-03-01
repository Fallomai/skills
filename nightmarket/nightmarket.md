# Nightmarket — Agent API Marketplace Instructions

You have access to Nightmarket, a marketplace where AI agents discover and pay for third-party API services. Every call settles on-chain in USDC via the x402 protocol.

**Marketplace:** `https://nightmarket.ai/marketplace`

---

## Setup — Choose one

### Option A: MCP Server (recommended for Claude Code)

Add this to your MCP configuration:

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
- `get_service_details` — Get full details, pricing, and request/response examples
- `call_service` — Call an API with automatic USDC payment

### Option B: REST API (works with any agent)

No MCP required. Call services directly through the Nightmarket proxy:

```
GET/POST https://nightmarket.ai/api/x402/<endpoint_id>
```

The proxy returns HTTP `402 Payment Required` with payment details. See the "Paying for calls" section below.

---

## Paying for calls

### With CrowPay (recommended — no private key needed)

If you have Crow set up, payments are automatic. When you call a Nightmarket service and get a `402`, Crow handles it:

1. Call the Nightmarket proxy URL
2. Receive `402 Payment Required` with a `PAYMENT-REQUIRED` header
3. Forward the 402 to Crow's `/authorize` endpoint — Crow checks spending rules and signs the payment
4. Retry your request with the `X-PAYMENT` header from Crow's response

No wallet private key needed in your config. Crow manages the wallet with spending rules and human approval thresholds.

**Don't have Crow yet?** Set it up:

```
curl -o crow.md https://raw.githubusercontent.com/Fallomai/skills/main/crow/crow.md
```

Read `crow.md` for full instructions. Get a wallet at https://crowpay.ai

### With a direct wallet

Set `WALLET_KEY` in your MCP server config (Option A above). The MCP server signs payments automatically using your wallet's private key.

Need a wallet? Get one at https://crowpay.ai

---

## Browsing services

### Via MCP

```
browse_services(search?: "weather")
```

Returns a list of available APIs with name, method, price, seller, and endpoint ID.

### Via REST

Browse the marketplace at https://nightmarket.ai/marketplace or use the Convex API directly.

---

## Calling a service

### Via MCP

```
call_service(
  endpoint_id: "<endpoint_id>",
  method: "POST",
  body: "<request body>"
)
```

The MCP server handles the full x402 payment flow automatically.

### Via REST (with Crow)

```
POST https://nightmarket.ai/api/x402/<endpoint_id>
Content-Type: application/json

<request body>
```

If you get `402 Payment Required`:

1. Read the `PAYMENT-REQUIRED` header from the response
2. Send it to Crow:
   ```
   POST https://api.crowpay.ai/authorize
   X-API-Key: crow_sk_...
   Content-Type: application/json

   {
     "paymentRequired": <the full 402 response body>,
     "merchant": "Nightmarket — <service name>",
     "reason": "API call to <service name>"
   }
   ```
3. On `200`, retry your original request with:
   ```
   X-PAYMENT: <base64(JSON.stringify(response_body))>
   ```
4. On `202`, poll `/authorize/status?id=<approvalId>` for human approval

---

## Decision tree

```
Want to call a Nightmarket API?
├── Have the MCP server set up?
│   └── Use call_service(endpoint_id, method, body)
│       └── Payment handled automatically
│
└── Calling via REST?
    ├── Have Crow?
    │   └── Call proxy → get 402 → forward to Crow → retry with payment
    │
    └── Have a wallet private key?
        └── Use the MCP server (Option A) — it handles signing
```

---

## Important details

- **Network**: Base (USDC settlement)
- **Protocol**: x402 — payment-per-request via HTTP headers
- **Currency**: USDC (on-chain)
- **Proxy URL pattern**: `https://nightmarket.ai/api/x402/<endpoint_id>`
- All prices are in USDC (e.g., `0.01` = one cent per call)
- The MCP server requires `WALLET_KEY` env var for direct wallet payment
- CrowPay users don't need `WALLET_KEY` — Crow manages the wallet
