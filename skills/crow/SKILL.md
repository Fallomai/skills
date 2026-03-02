---
name: crow
description: Agent payment service via CrowPay. Use when your agent needs to pay for APIs (x402/USDC) or merchants (credit card), handle HTTP 402 Payment Required responses, set up a wallet with spending rules, or pay for anything on the internet. Use this skill whenever the user mentions payments, wallets, spending, buying API credits, paying merchants, 402 errors, USDC, or agent commerce — even if they don't mention "Crow" by name.
---

# Crow — Agent Payment Service

Crow lets your AI agent pay for APIs and services autonomously — within spending rules set by the wallet owner. Supports x402 (USDC on Base) and credit card payments via Stripe.

**Base URL:** `https://api.crowpay.ai`
**Dashboard:** https://crowpay.ai/dashboard

## When to Use This Skill

- Agent encounters HTTP `402 Payment Required`
- Agent needs to pay a merchant (API credits, subscriptions, etc.)
- User wants to set up an agent wallet with spending rules
- User asks about CrowPay, Crow, or agent payments
- User wants to buy something or pay for an API

## Quick Setup

### 1. Create a wallet and API key

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

**IMPORTANT:** Save the `apiKey` immediately — it is shown only once and cannot be retrieved later.

### 2. Claim the wallet

Tell the user to visit the `claimUrl` to create an account and set spending rules.

### 3. Fund the wallet

- Send USDC on Base to the `walletAddress`, OR
- Add a credit card in the dashboard at https://crowpay.ai/dashboard

### 4. Authenticate all subsequent requests

All endpoints except `/setup` require the API key header:
```
X-API-Key: crow_sk_...
```

## Decision Tree

```
Need to pay?
├── Got a 402 from an API?
│   └── POST /authorize with the 402 body
│       ├── 200 → Retry original request with X-PAYMENT header
│       ├── 202 → Poll GET /authorize/status for human approval
│       └── 403 → Blocked by spending rules, inform user
│
└── Need to pay a merchant with credit card?
    └── POST /authorize/card
        ├── 200 → Use returned sptToken to pay merchant
        ├── 202 → Poll GET /authorize/status for human approval
        └── 403 → Blocked by spending rules, inform user
```

---

## Endpoints

### POST /setup

Create a new agent wallet and API key. No authentication required.

```
POST https://api.crowpay.ai/setup
Content-Type: application/json

{
  "network": "eip155:8453"     // optional, defaults to Base mainnet
}
```

**200 OK:**
```json
{
  "apiKey": "crow_sk_...",
  "walletAddress": "0x...",
  "claimUrl": "https://crowpay.ai/claim/...",
  "fundingInstructions": "Send USDC on Base to 0x..."
}
```

**429:** Rate limited. Wait and retry.

---

### POST /authorize

Forward an HTTP 402 Payment Required response to get a signed USDC payment authorization.

```
POST https://api.crowpay.ai/authorize
X-API-Key: crow_sk_...
Content-Type: application/json

{
  "paymentRequired": {
    "x402Version": 2,
    "resource": { "url": "https://api.example.com/resource" },
    "accepts": [{
      "scheme": "exact",
      "network": "eip155:8453",
      "amount": "1000000",
      "asset": "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913",
      "payTo": "0x...",
      "maxTimeoutSeconds": 60,
      "extra": { "name": "USDC", "version": "2" }
    }]
  },
  "merchant": "Name of the API/service",
  "reason": "Why you need this payment"
}
```

Pass the **entire 402 response body** as `paymentRequired`. Provide clear `merchant` and `reason` values — the wallet owner sees these when approving.

**200 OK — Auto-approved:**
```json
{
  "x402Version": 2,
  "resource": { "url": "https://api.example.com/resource" },
  "accepted": {
    "scheme": "exact",
    "network": "eip155:8453",
    "amount": "1000000",
    "asset": "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913",
    "payTo": "0x...",
    "maxTimeoutSeconds": 60,
    "extra": { "name": "USDC", "version": "2" }
  },
  "payload": {
    "signature": "0x...",
    "authorization": {
      "from": "0x...",
      "to": "0x...",
      "value": "1000000",
      "validAfter": "1740672089",
      "validBefore": "1740672154",
      "nonce": "0x..."
    }
  }
}
```

To retry the original request, base64-encode the full JSON response and send it as the `X-PAYMENT` header:
```
X-PAYMENT: <base64(JSON.stringify(response_body))>
```

**202 Accepted — Needs human approval:**
```json
{
  "status": "pending",
  "approvalId": "...",
  "expiresAt": 1234567890,
  "message": "Payment requires human approval. Poll GET /authorize/status?id=..."
}
```

**403 Forbidden:** Payment denied by spending rules. Do not retry with the same parameters.

**401 Unauthorized:** Invalid or missing API key.

---

### POST /authorize/card

Request a credit card payment via Stripe Shared Payment Token.

```
POST https://api.crowpay.ai/authorize/card
X-API-Key: crow_sk_...
Content-Type: application/json

{
  "amountCents": 1000,                    // required — $10.00
  "currency": "usd",                      // optional, defaults to "usd"
  "merchant": "OpenAI",                   // required
  "reason": "GPT-4 API credits",          // required
  "merchantStripeAccount": "acct_...",    // optional — Stripe Connect account
  "paymentMethodId": "pm_..."            // optional — uses default card if omitted
}
```

**200 OK — Auto-approved:**
```json
{
  "approved": true,
  "sptToken": "spt_...",
  "transactionId": "..."
}
```

Use the `sptToken` to pay the merchant via Stripe. Token expires in 1 hour.

**202 Accepted — Needs human approval:**
```json
{
  "status": "pending",
  "approvalId": "...",
  "expiresAt": 1234567890,
  "message": "Payment requires human approval. Poll GET /authorize/status?id=..."
}
```

**403 Forbidden:** Payment denied by spending rules.

**400 Bad Request:** Missing required fields or invalid amount.

---

### GET /authorize/status

Poll for the status of a pending approval.

```
GET https://api.crowpay.ai/authorize/status?id=<approvalId>
X-API-Key: crow_sk_...
```

Poll every **3 seconds**. Do not poll faster.

**Possible responses:**

| Status | Meaning | Action |
|--------|---------|--------|
| `"pending"` | Waiting for human | Keep polling |
| `"signing"` | Approved, generating payload | Keep polling |
| Response with `payload` field | Ready | Use the signed payload |
| `"denied"` | Owner rejected | Stop, inform user |
| `"timeout"` | Approval window expired | Stop, inform user |
| `"failed"` | Error during signing | Stop, inform user |

When the response contains a full payload (with `x402Version`, `accepted`, `payload` fields), use it the same way as a 200 from `/authorize`.

For card payments, the response will contain `sptToken` when approved.

---

### POST /settle

Report that an x402 payment has settled on-chain. Idempotent — safe to call multiple times.

```
POST https://api.crowpay.ai/settle
X-API-Key: crow_sk_...
Content-Type: application/json

{
  "transactionId": "...",
  "txHash": "0x..."
}
```

**200 OK:**
```json
{
  "success": true
}
```

Or if already settled:
```json
{
  "success": true,
  "alreadySettled": true
}
```

---

## Important Numbers

| Type | Format | Example | Value |
|------|--------|---------|-------|
| USDC (x402) | Atomic units, 6 decimals | `1000000` | $1.00 |
| USDC (x402) | Atomic units, 6 decimals | `100000` | $0.10 |
| Card | Cents | `100` | $1.00 |
| Card | Cents | `1000` | $10.00 |

- **Network:** Base mainnet (`eip155:8453`)
- **USDC contract:** `0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913`

## Default Spending Rules (when wallet is claimed)

- Per-transaction limit: $25
- Daily limit: $50
- Auto-approve threshold: $5 — payments above this need human approval
- Owners can customize all limits in the dashboard

## References

Read these for deeper details on specific flows:

- `references/x402-flow.md` — Step-by-step 402 payment flow with retry logic
- `references/card-payments.md` — Credit card payment flow and SPT usage
- `references/error-handling.md` — All error codes, status codes, and retry behavior

## Finding Services to Pay For

Use [Nightmarket](https://nightmarket.ai) to discover paid APIs your agent can call. Every Nightmarket service uses x402 — Crow handles the payments automatically.

Install the Nightmarket skill:
```
npx skills add https://github.com/Fallomai/skills --skill nightmarket
```
