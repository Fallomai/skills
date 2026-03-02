# Credit Card Payments

Pay merchants with a credit card using Stripe Shared Payment Tokens (SPTs).

## Overview

When no 402 is involved (e.g., paying for SaaS, API credits, subscriptions), use the card payment flow. Crow provisions a scoped Stripe Shared Payment Token that the agent uses to pay the merchant.

## Prerequisites

- Wallet must be claimed (user visited `claimUrl`)
- A credit card must be added in the dashboard at https://crowpay.ai/dashboard
- Card spending rules are auto-created when a card is added

## Request

```
POST https://api.crowpay.ai/authorize/card
X-API-Key: crow_sk_...
Content-Type: application/json

{
  "amountCents": 1000,
  "currency": "usd",
  "merchant": "OpenAI",
  "reason": "GPT-4 API credits",
  "merchantStripeAccount": "acct_...",
  "paymentMethodId": "pm_..."
}
```

### Fields

| Field | Required | Description |
|-------|----------|-------------|
| `amountCents` | Yes | Amount in cents. `1000` = $10.00 |
| `currency` | No | Defaults to `"usd"` |
| `merchant` | Yes | Human-readable merchant name |
| `reason` | Yes | Why the payment is needed |
| `merchantStripeAccount` | No | Stripe Connect account ID if applicable |
| `paymentMethodId` | No | Specific card to use. Uses default card if omitted |

## Responses

### 200 — Auto-approved

```json
{
  "approved": true,
  "sptToken": "spt_...",
  "transactionId": "..."
}
```

The `sptToken` is a Stripe Shared Payment Token. Use it to pay the merchant through Stripe's payment API. Token expires in **1 hour**.

### 202 — Pending human approval

Amount exceeds the auto-approve threshold (default: $5). The wallet owner must approve.

```json
{
  "status": "pending",
  "approvalId": "...",
  "expiresAt": 1234567890,
  "message": "Payment requires human approval. Poll GET /authorize/status?id=..."
}
```

Poll `/authorize/status?id=<approvalId>` every 3 seconds. When approved, the response will contain `sptToken`.

### 403 — Denied

Spending rules blocked the payment:
- Per-transaction limit exceeded (default: $25)
- Daily limit exceeded (default: $50)
- Merchant blacklisted or not on whitelist

### 400 — Bad request

- `amountCents` is missing or not a positive integer
- `merchant` or `reason` is missing
- No payment methods configured (user needs to add a card first)

## Default card spending rules

Created automatically when a card is added to the dashboard:

| Rule | Default |
|------|---------|
| Daily limit | $50 (5000 cents) |
| Per-transaction limit | $25 (2500 cents) |
| Auto-approve threshold | $5 (500 cents) |

Owners can customize these in the dashboard under the "Rules" tab.

## Complete example

```javascript
const response = await fetch("https://api.crowpay.ai/authorize/card", {
  method: "POST",
  headers: {
    "X-API-Key": "crow_sk_...",
    "Content-Type": "application/json"
  },
  body: JSON.stringify({
    amountCents: 500,
    merchant: "OpenAI",
    reason: "GPT-4 API credits for code review"
  })
});

if (response.status === 200) {
  const { sptToken } = await response.json();
  // Use sptToken with Stripe to pay merchant
} else if (response.status === 202) {
  const { approvalId } = await response.json();
  // Poll /authorize/status for approval, then get sptToken
}
```

## Settlement

Card payments are tracked automatically via Stripe webhooks:
- `shared_payment.issued_token.used` → transaction marked as settled
- `shared_payment.issued_token.deactivated` → transaction marked as failed

No need to call `/settle` for card payments.
