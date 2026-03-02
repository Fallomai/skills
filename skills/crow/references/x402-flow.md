# x402 Payment Flow (USDC on Base)

Step-by-step guide for handling HTTP 402 Payment Required responses using Crow.

## Overview

x402 is a protocol where APIs return HTTP 402 with payment details. Crow checks spending rules, signs an EIP-3009 authorization, and returns a payment payload your agent uses to retry the request.

## Step 1: Receive the 402

When you call an x402-enabled API, you get back:

```
HTTP/1.1 402 Payment Required
Content-Type: application/json

{
  "x402Version": 2,
  "resource": {
    "url": "https://api.example.com/v1/data",
    "description": "Access to data endpoint"
  },
  "accepts": [
    {
      "scheme": "exact",
      "network": "eip155:8453",
      "amount": "1000000",
      "asset": "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913",
      "payTo": "0xRecipientAddress",
      "maxTimeoutSeconds": 60,
      "extra": {
        "name": "USDC",
        "version": "2"
      }
    }
  ]
}
```

## Step 2: Forward to Crow

Pass the entire 402 body to Crow's `/authorize` endpoint:

```
POST https://api.crowpay.ai/authorize
X-API-Key: crow_sk_...
Content-Type: application/json

{
  "paymentRequired": <entire 402 response body>,
  "merchant": "ExampleAPI",
  "reason": "Fetching data for user request"
}
```

**Required fields:**
- `paymentRequired` — the full 402 response JSON
- `merchant` — human-readable name of the API/service (wallet owner sees this)
- `reason` — why the payment is needed (wallet owner sees this)

## Step 3: Handle the response

### 200 — Auto-approved

Crow checked spending rules and the amount is within the auto-approve threshold.

The response is a signed payment payload. Retry your original request with this payload base64-encoded in the `X-PAYMENT` header:

```javascript
const paymentPayload = JSON.stringify(crowResponse);
const encoded = btoa(paymentPayload);

// Retry the original request
fetch("https://api.example.com/v1/data", {
  headers: {
    "X-PAYMENT": encoded
  }
});
```

### 202 — Pending human approval

The amount exceeds the auto-approve threshold. The wallet owner must approve in their dashboard.

```json
{
  "status": "pending",
  "approvalId": "abc123",
  "expiresAt": 1234567890,
  "message": "Payment requires human approval. Poll GET /authorize/status?id=abc123"
}
```

Poll for the decision:

```
GET https://api.crowpay.ai/authorize/status?id=abc123
X-API-Key: crow_sk_...
```

- Poll every **3 seconds**, not faster
- When status is `"pending"` or `"signing"` → keep polling
- When response contains a `payload` field → use it (same as 200 flow above)
- When status is `"denied"`, `"timeout"`, or `"failed"` → stop and inform user

### 403 — Denied

Spending rules blocked this payment. Possible reasons:
- Amount exceeds per-transaction limit
- Daily spending limit reached
- Merchant is blacklisted
- Merchant is not on the whitelist (if whitelist is enabled)

**Do not retry** with the same parameters. Inform the user.

### 401 — Unauthorized

API key is missing or invalid. Check your `X-API-Key` header.

### 400 — Bad request

Missing required fields or incompatible payment option. Check:
- `paymentRequired` is present and valid
- `merchant` and `reason` are provided
- The wallet supports the requested network/asset

## Step 4: Report settlement

After the x402 facilitator settles payment on-chain, report it:

```
POST https://api.crowpay.ai/settle
X-API-Key: crow_sk_...
Content-Type: application/json

{
  "transactionId": "...",
  "txHash": "0x..."
}
```

This is idempotent — safe to call multiple times.

## Complete example

```javascript
// 1. Call the API
let response = await fetch("https://api.example.com/v1/data");

if (response.status === 402) {
  const paymentRequired = await response.json();

  // 2. Forward to Crow
  const crowResponse = await fetch("https://api.crowpay.ai/authorize", {
    method: "POST",
    headers: {
      "X-API-Key": "crow_sk_...",
      "Content-Type": "application/json"
    },
    body: JSON.stringify({
      paymentRequired,
      merchant: "ExampleAPI",
      reason: "Fetching data for user request"
    })
  });

  if (crowResponse.status === 200) {
    const payload = await crowResponse.json();
    const encoded = btoa(JSON.stringify(payload));

    // 3. Retry with payment
    response = await fetch("https://api.example.com/v1/data", {
      headers: { "X-PAYMENT": encoded }
    });
  } else if (crowResponse.status === 202) {
    const { approvalId } = await crowResponse.json();

    // 3. Poll for approval
    let approved = false;
    while (!approved) {
      await new Promise(r => setTimeout(r, 3000));
      const status = await fetch(
        `https://api.crowpay.ai/authorize/status?id=${approvalId}`,
        { headers: { "X-API-Key": "crow_sk_..." } }
      ).then(r => r.json());

      if (status.payload) {
        const encoded = btoa(JSON.stringify(status));
        response = await fetch("https://api.example.com/v1/data", {
          headers: { "X-PAYMENT": encoded }
        });
        approved = true;
      } else if (["denied", "timeout", "failed"].includes(status.status)) {
        throw new Error(`Payment ${status.status}: ${status.reason}`);
      }
    }
  } else if (crowResponse.status === 403) {
    const { reason } = await crowResponse.json();
    throw new Error(`Payment denied: ${reason}`);
  }
}
```

## Key details

- USDC amounts are in **atomic units** with 6 decimals: `1000000` = $1.00
- Network is always Base mainnet: `eip155:8453`
- USDC contract: `0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913`
- Crow signs EIP-3009 `TransferWithAuthorization` — the facilitator settles on-chain
- The wallet owner's private key never leaves Crow's server
