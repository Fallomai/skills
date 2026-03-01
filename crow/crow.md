# Crow — Agent Payment Instructions

You have access to Crow, a payment service that lets you spend money on behalf of your operator. Crow manages a wallet with spending rules — you make payment requests, Crow checks the rules, and either approves or denies.

**Base URL:** `https://api.crowpay.ai`

---

## Setup

If you don't have a Crow API key yet, create one:

```
POST https://api.crowpay.ai/setup
Content-Type: application/json

{}
```

Response:
```json
{
  "apiKey": "crow_sk_...",
  "walletAddress": "0x...",
  "claimUrl": "https://crowpay.ai/claim/...",
  "fundingInstructions": "Send USDC on Base to 0x..."
}
```

**Save the `apiKey` immediately — it is only shown once.** All subsequent requests require the header `X-API-Key: crow_sk_...`.

Tell the user to visit the `claimUrl` to claim the wallet and configure spending rules. They also need to fund the wallet with USDC on Base or add a credit card.

---

## When You Get a 402 (x402 USDC Payment)

If you call an API and receive HTTP `402 Payment Required`, follow these steps:

### 1. Forward the 402 to Crow

```
POST https://api.crowpay.ai/authorize
X-API-Key: crow_sk_...
Content-Type: application/json

{
  "paymentRequired": <the full 402 response body>,
  "merchant": "Name of the API/service",
  "reason": "Why you need this"
}
```

Pass the entire 402 response JSON as `paymentRequired`. Provide a clear `merchant` name and `reason` — the wallet owner sees these.

### 2. Handle the response

**If 200 — Approved.** You get a signed payment payload. Retry your original request with the payload:

```
base64(JSON.stringify(response_body))
```

Put this in the `X-PAYMENT` header of your retry request.

**If 202 — Pending human approval.** The amount exceeded the auto-approve threshold. Poll for a decision:

```
GET https://api.crowpay.ai/authorize/status?id=<approvalId>
X-API-Key: crow_sk_...
```

Poll every 3 seconds. When the response contains a `payload` field, use it. If `status` is `"denied"`, `"timeout"`, or `"failed"`, stop — the payment was rejected.

**If 403 — Denied.** The spending rules blocked this payment. Do not retry with the same parameters.

### 3. After successful payment

Report the settlement back to Crow:

```
POST https://api.crowpay.ai/settle
X-API-Key: crow_sk_...
Content-Type: application/json

{
  "transactionId": "...",
  "txHash": "0x..."
}
```

---

## Paying with Credit Card

If you need to pay a merchant with a credit card (no 402 involved):

```
POST https://api.crowpay.ai/authorize/card
X-API-Key: crow_sk_...
Content-Type: application/json

{
  "amountCents": 1000,
  "merchant": "OpenAI",
  "reason": "GPT-4 API credits"
}
```

- `amountCents` is in cents: 1000 = $10.00
- `currency` is optional, defaults to `"usd"`

**If 200 — Approved.** Use the returned `sptToken` to pay the merchant via Stripe. The token expires in 1 hour.

**If 202 — Pending.** Poll `/authorize/status?id=<approvalId>` same as above.

**If 403 — Denied.** The spending rules blocked this payment.

---

## Decision Tree

```
Need to pay?
├── Got a 402 from an API?
│   └── POST /authorize with the 402 body
│       ├── 200 → Retry original request with X-PAYMENT header
│       ├── 202 → Poll /authorize/status until resolved
│       └── 403 → Payment blocked, inform the user
│
└── Need to pay a merchant with a card?
    └── POST /authorize/card
        ├── 200 → Use sptToken to pay
        ├── 202 → Poll /authorize/status until resolved
        └── 403 → Payment blocked, inform the user
```

---

## Error Handling

| Status | Meaning | What to Do |
|--------|---------|------------|
| 400 | Bad request / missing fields | Fix your request |
| 401 | Invalid API key | Check your key, do not retry |
| 403 | Denied by spending rules | Do not retry with same params |
| 429 | Rate limited | Wait and retry |
| 5xx | Server error | Retry with backoff |

---

## Important Details

- **USDC amounts** are in atomic units (6 decimals): `1000000` = $1.00
- **Card amounts** are in cents: `100` = $1.00
- **Network**: Base mainnet (`eip155:8453`)
- **USDC address**: `0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913`
- Poll `/authorize/status` every 3 seconds, not faster
- Always provide descriptive `merchant` and `reason` values — the wallet owner sees them
- The `/settle` endpoint is idempotent — safe to call multiple times
