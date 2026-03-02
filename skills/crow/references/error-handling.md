# Error Handling

All Crow API error responses follow this format:

```json
{
  "error": "Short error message",
  "reason": "Detailed explanation (sometimes)",
  "details": "Technical details (sometimes)"
}
```

## HTTP Status Codes

| Status | Meaning | Retry? | Action |
|--------|---------|--------|--------|
| 200 | Success | — | Proceed with response |
| 202 | Pending approval | Poll | Poll `/authorize/status` every 3s |
| 400 | Bad request | No | Fix the request body/params |
| 401 | Unauthorized | No | Check API key is valid |
| 403 | Forbidden / Denied | No | Spending rules blocked it — inform user |
| 404 | Not found | No | Check the resource ID |
| 422 | Invalid state | No | Transaction is in wrong state for this action |
| 429 | Rate limited | Yes | Wait, then retry with backoff |
| 500 | Server error | Yes | Retry with exponential backoff |

## Common Errors by Endpoint

### POST /setup

| Error | Cause |
|-------|-------|
| `429 Rate limit exceeded` | Too many setup calls from this IP |

### POST /authorize

| Error | Cause |
|-------|-------|
| `400 Missing required fields: paymentRequired, merchant, reason` | Request body incomplete |
| `400 No compatible payment option` | Wallet doesn't support the requested network/asset |
| `401 Missing X-API-Key header` | No API key provided |
| `401 Invalid API key or wallet not found` | Key is wrong or revoked |
| `401 Wallet not claimed` | User hasn't visited the claim URL yet |
| `403 Payment denied` | Spending rules blocked the payment |

### POST /authorize/card

| Error | Cause |
|-------|-------|
| `400 amountCents must be a positive integer` | Invalid amount |
| `400 Missing required fields: merchant, reason` | Request body incomplete |
| `400 No payment methods configured` | No card added in dashboard |
| `403 Payment denied` | Spending rules blocked the payment |
| `403 Payment method not found or not owned by you` | Wrong `paymentMethodId` |

### GET /authorize/status

| Error | Cause |
|-------|-------|
| `400 Missing id query parameter` | No `?id=` in URL |
| `403 Forbidden` | Approval belongs to a different wallet |
| `404 Approval not found` | Invalid approval ID |

### POST /settle

| Error | Cause |
|-------|-------|
| `400 Missing transactionId or txHash` | Request body incomplete |
| `403 Forbidden` | Transaction belongs to a different wallet |
| `404 Transaction not found` | Invalid transaction ID |
| `422 Cannot settle transaction with status: ...` | Transaction is pending approval or already failed |

## Retry Strategy

For retryable errors (429, 5xx):

```javascript
async function withRetry(fn, maxRetries = 3) {
  for (let i = 0; i < maxRetries; i++) {
    try {
      return await fn();
    } catch (err) {
      if (i === maxRetries - 1) throw err;
      const delay = Math.min(1000 * Math.pow(2, i), 10000);
      await new Promise(r => setTimeout(r, delay));
    }
  }
}
```

## Polling Best Practices

When polling `/authorize/status`:
- Poll every **3 seconds** — not faster
- Set a max poll duration (e.g., 5 minutes) to avoid infinite loops
- Handle all terminal states: `denied`, `timeout`, `failed`
- The `expiresAt` timestamp tells you when the approval window closes

## Key Constants

| Constant | Value |
|----------|-------|
| USDC contract (Base) | `0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913` |
| Network (CAIP-2) | `eip155:8453` |
| USDC decimals | 6 |
| $1.00 in USDC atomic | `1000000` |
| $1.00 in card cents | `100` |
