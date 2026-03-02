# Nightmarket API Reference

All Nightmarket services are called via standard HTTP requests to the proxy. No SDK, no special client — just curl.

## Proxy URL

```
<METHOD> https://nightmarket.ai/api/x402/<endpoint_id>
```

- `METHOD`: GET, POST, PUT, PATCH, or DELETE (must match what the service expects)
- `endpoint_id`: the service's unique ID (find it on the marketplace or via browse)

## Request Headers

| Header | Required | Description |
|--------|----------|-------------|
| `Content-Type` | For POST/PUT/PATCH | Usually `application/json` |
| `Authorization` | If service requires it | Passed through to the seller's API |
| `payment-signature` | After 402 | Signed x402 payment proof |

## Request Body

Pass the body exactly as the service expects it. The proxy forwards it unchanged to the seller's API.

## Response Flow

### First call → 402 Payment Required

Every first call returns 402. This is normal — it's how x402 works.

**Response headers:**
- `PAYMENT-REQUIRED`: encoded payment requirements containing:
  - `scheme`: "exact"
  - `payTo`: seller's payment address (on Base)
  - `price`: amount in USDC (e.g., "$0.01")
  - `network`: "base"

**Response body:** `{}`

### Retry with payment → Success

After paying, retry the exact same request with the payment proof.

**Request header to add:**
- `payment-signature`: your signed payment (or `X-PAYMENT` if using CrowPay)

**Successful response headers:**
- `PAYMENT-RESPONSE`: settlement proof containing `txHash` (on-chain transaction hash)

**Successful response body:** the seller's API response, passed through unchanged.

## Error Codes

| Status | Meaning | Action |
|--------|---------|--------|
| 200-299 | Success | Response body is the API result |
| 400 | Invalid endpoint ID or bad payment signature | Check your endpoint_id and payment format |
| 402 | Payment required | Normal — sign payment and retry |
| 402 (after payment) | Payment verification or settlement failed | Check wallet balance, verify payment was signed correctly |
| 403 | Endpoint URL blocked | Internal safety check — the seller's URL is not allowed |
| 404 | Endpoint not found or inactive | Service may have been removed or deactivated |
| 502 | Seller's API unreachable | The seller's backend is down — try again later |
| 503 | Seller payment not configured | Seller hasn't set up their payout wallet yet |

## Complete Examples

### GET request

```bash
# First call — get payment requirements
curl -i -X GET "https://nightmarket.ai/api/x402/abc123?city=NYC"
# HTTP/1.1 402 Payment Required
# PAYMENT-REQUIRED: <encoded payment details>

# Retry with payment
curl -X GET "https://nightmarket.ai/api/x402/abc123?city=NYC" \
  -H "payment-signature: <signed payment>"
# HTTP/1.1 200 OK
# PAYMENT-RESPONSE: <settlement proof with txHash>
# {"temp": 72, "conditions": "sunny", "forecast": [...]}
```

### POST request with body

```bash
# First call
curl -i -X POST "https://nightmarket.ai/api/x402/def456" \
  -H "Content-Type: application/json" \
  -d '{"query": "sentiment analysis", "text": "I love this product"}'
# HTTP/1.1 402 Payment Required

# Retry with payment
curl -X POST "https://nightmarket.ai/api/x402/def456" \
  -H "Content-Type: application/json" \
  -H "payment-signature: <signed payment>" \
  -d '{"query": "sentiment analysis", "text": "I love this product"}'
# HTTP/1.1 200 OK
# {"sentiment": "positive", "confidence": 0.95}
```

## Finding Endpoint IDs

Browse the marketplace at https://nightmarket.ai/marketplace. Each service listing shows:
- **Endpoint ID** — what you pass in the URL
- **HTTP method** — GET, POST, etc.
- **Price** — USDC per call
- **Request/response examples** — what to send and what you get back
- **Total calls & revenue** — usage stats to gauge reliability

## Rate Limits & Timeouts

- **Proxy timeout:** 30 seconds per request
- **No rate limits** from Nightmarket itself (individual sellers may have their own)
- **Payment:** each call is independent — no sessions or tokens to manage
