---
layout: default
title: Integration Guide
---

# Integration Guide

This page covers everything needed to build against the DreamPay merchant API: authentication, conventions, the invoice and payout endpoints, webhook delivery and signature verification, and error handling.

Base URLs:

| Environment | Base URL |
|---|---|
| Production | `https://api.smartcardpro.com` |
| Sandbox | `https://api-sandbox.dreampay.cards` |

All examples below assume `$DREAMPAY_BASE_URL` and `$DREAMPAY_API_KEY` are set to the values for your environment.

## 2. Integration principles

### Authentication

Every merchant API request carries your API key in the **`x-api-key`** header:

```
x-api-key: <your API key>
```

- A missing header returns `401` with `"error": "api key is empty"`.
- An unknown, revoked, or expired key returns `403` with `"error": "access forbidden"`.
- The API key **is** your merchant identity — no request body or route parameter ever carries a merchant ID. Every invoice, payout, and balance you see is scoped to the key you authenticated with.
- Treat your API key like a password: keep it in a secret manager, never embed it in client-side code, and rotate it if it may have been exposed. DreamPay supports issuing multiple active keys per merchant so you can roll a new one in before revoking the old one.

### Response envelope

Every response uses one envelope:

```json
{
  "data": { },
  "metadata": {
    "status_code": 200,
    "error": "only present on errors",
    "message": "optional validation detail",
    "timestamp": "2026-08-12T10:00:00Z"
  }
}
```

On a `5xx`, the response body is always sanitized to `"error": "internal server error"` — no internal detail is ever returned to a merchant-authenticated caller.

### Conventions

- **Amounts are JSON strings** (`"amount": "25.5"`). Parse them with a decimal type in your language — never as a floating-point number. Percentages are also decimal strings (`"1.5"` = 1.5%).
- **Timestamps** are RFC 3339, UTC.
- **Tickers** are uppercase (`USDT`, not `usdt`). Send them uppercase on every request.
- **`chain_id`** is the numeric chain ID for the network the balance lives on (see the [Overview](index.html) for known values). Always confirm current supported chains via the tickers endpoints below rather than hard-coding them.
- **Pagination**: list endpoints require both `page` and `pageSize` query parameters (positive integers) and return a `total_count` alongside the results. Results are ordered newest-first.
- **Idempotency**: see below — always send an idempotency key on writes.

### Idempotency

- **Invoices**: `idempotency_key` is required and must be unique per merchant. The first call with a given key returns `201` with the new invoice. Replaying the same key returns `200` with the invoice as it was originally created — any different fields in the replay are ignored.
- **Payouts**: `idempotency_key` is optional but strongly recommended. Without one, every retried call creates a new payout. With one, the first call returns `201`, replays return `200` with the original payout. Always generate and persist your idempotency key *before* calling, and retry with the same key on timeouts.

## 3. API reference

Full machine-readable spec: the Swagger UI for your environment (linked above). This section covers the endpoints you'll use most.

### 3.1 Invoices

An invoice is a request for an exact amount in one `(ticker, chain_id)`, paid by a user in the DreamPay app.

**Create an invoice**

```bash
curl -X POST "$DREAMPAY_BASE_URL/v1/invoices" \
  -H "x-api-key: $DREAMPAY_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "idempotency_key": "order-10345",
    "client_transaction_id": "order-10345",
    "chain_id": 137,
    "amount": "25.50",
    "ticker": "USDT",
    "issued_at": "2026-08-12T10:00:00Z",
    "expires_at": "2026-08-12T10:30:00Z"
  }'
```

| Field | Notes |
|---|---|
| `idempotency_key` | required; unique per merchant |
| `client_transaction_id` | required; your own reference — echoed back on the invoice and in webhooks |
| `chain_id` | required |
| `amount` | required, > 0 — the gross amount the payer will be charged |
| `ticker` | required, uppercase; must currently accept invoice payments |
| `issued_at` | required by validation, but the server always stamps its own creation time |
| `expires_at` | required. Validate this on your side before sending — set a sensible window for your use case |

Response (`201` on first creation, `200` on idempotent replay):

```json
{
  "data": {
    "id": "0b6c9f0e-2f43-4b8e-9f7a-1c2d3e4f5a6b",
    "merchant_id": "<your merchant id>",
    "user_id": null,
    "idempotency_key": "order-10345",
    "client_transaction_id": "order-10345",
    "amount": "25.5",
    "chain_id": 137,
    "ticker": "USDT",
    "platform_fee_percent": "1",
    "cashback_percent": "0.5",
    "platform_fee_amount": "0.255",
    "cashback_amount": "0.1275",
    "net_amount": "25.1175",
    "issued_at": "2026-08-12T10:00:01Z",
    "expires_at": "2026-08-12T10:30:00Z",
    "paid_at": null,
    "invoice_status": "pending",
    "invoice_sub_status": "waiting_for_payment",
    "created_at": "2026-08-12T10:00:01Z",
    "updated_at": "2026-08-12T10:00:01Z"
  },
  "metadata": { "status_code": 201, "timestamp": "2026-08-12T10:00:01Z" }
}
```

`net_amount` is what actually lands on your merchant balance once the invoice is paid (`amount` minus the platform fee and any cashback funded from your account — see your fee configuration via `GET /v1/merchant/fees`). Reconcile your accounting against `net_amount`, not `amount`.

Once an invoice is paid, the response also includes a `p2p_transaction` object describing the underlying settlement.

**Deliver the invoice to your customer** through your own channel (a deeplink into the DreamPay app, a QR code, etc.) using the invoice `id`. Treat the invoice ID as a bearer link to that payment request — share it only with the intended payer, over a channel you control.

**Lifecycle**: `pending → completed` (paid) or `pending → rejected` (payer rejects, or you cancel it while still unclaimed — see below). There is no `expired` status and **no webhook fires on expiry** — see [Webhooks](#4-webhooks) for how to handle that.

**Cancel an invoice** (only while pending and not yet claimed by a payer):

```bash
curl -X POST "$DREAMPAY_BASE_URL/v1/invoices/{id}/reject" \
  -H "x-api-key: $DREAMPAY_API_KEY"
```

Returns `200` with the invoice now `rejected`, and enqueues an `invoice.rejected` webhook. Returns `409` if a payer has already claimed the invoice (paid or rejected it).

**List / get invoices**

```
GET /v1/invoices?page=1&pageSize=20&status=pending
GET /v1/invoices/{id}
```

Optional filters on the list endpoint: `status` (`pending`, `completed`, `rejected`), `user_id`, `ticker`, `chain_id`.

### 3.2 Payouts

`POST /v1/merchant/payouts` moves funds from your merchant balance to a DreamPay app user.

```bash
curl -X POST "$DREAMPAY_BASE_URL/v1/merchant/payouts" \
  -H "x-api-key: $DREAMPAY_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "user_id": "<recipient user id>",
    "ticker": "USDT",
    "chain_id": 137,
    "amount": "10",
    "idempotency_key": "payout-771",
    "client_transaction_id": "payout-771"
  }'
```

Response (`201` on first creation, `200` on idempotent replay):

```json
{
  "data": {
    "id": 42,
    "sender_id": "<your merchant id>",
    "receiver_id": "<recipient user id>",
    "ticker": "USDT",
    "chain_id": 137,
    "gross_amount": "10",
    "service_fee": "0.1",
    "usd_xrate": "1",
    "type": "merchant_payout",
    "created_at": "...",
    "updated_at": "..."
  }
}
```

The amount actually credited to the recipient is `gross_amount − service_fee` (the payout object doesn't carry a separate `net_amount` field the way an invoice does). You're debited the gross amount; any payout fee configured on your account is deducted from what the recipient receives.

Before paying out, you can check whether a user ID exists with `GET /v1/merchant/users/{user_id}` (returns `{ "exists": true|false }`). Use this as a lightweight pre-check only — the payout call itself is the authoritative check for whether a payout can actually be completed.

`GET /v1/merchant/payouts` and `GET /v1/merchant/payouts/{id}` list and fetch payouts, with the same pagination and filtering conventions as invoices.

### 3.3 Balances, tickers, and rates

```
GET /v1/merchant/balances
```
Returns your current ledger balance per `(ticker, chain_id)`.

```
GET /v1/merchant/available-invoice-tickers
GET /v1/merchant/available-payout-tickers
```
Returns the `(ticker, chain)` pairs currently usable for each flow. **Check `is_active: true`** on each entry before offering it to a payer — the list can include entries that are temporarily disabled.

```
GET /v1/merchant/exchange-rates/{ticker}?quote=USD
```
Returns an indicative exchange rate for display purposes only (`quote` is `USD` or `EUR`). Invoices and payouts always settle in the exact ticker amount you specify — there's no FX conversion applied at settlement.

```
GET /v1/merchant/fees
```
Returns your current fee and cashback configuration (percentages), set by DreamPay when your account is configured. Contact your account manager to change it.

## 4. Webhooks

DreamPay sends webhooks to your configured `webhook_url` when an invoice's status changes.

### Events

| Event | Fired when |
|---|---|
| `invoice.completed` | A payer pays the invoice |
| `invoice.rejected` | A payer rejects the invoice, or you cancel it while unclaimed |

There is **no event for invoice creation** and **no event for expiry** — expiry is enforced only when a payer attempts to pay an expired invoice. If you haven't heard anything about an invoice by its `expires_at`, poll for it (see [Reconciliation](#reconciliation)).

There's also no top-level `event_type` field — distinguish the two events by `invoice_status` (`completed` vs `rejected`) and the presence of a `p2p_transaction` object (present only on `invoice.completed`).

Your webhook URL must be `https://` and is configured for you by the DreamPay team as part of onboarding. Ask your account manager to update it if it changes.

### Request format

```
POST <your webhook_url> HTTP/1.1
Content-Type: application/json
X-Invoice-Timestamp: 1754944000
X-Invoice-Key-Id: <kid>
X-Invoice-Signature: v1=<base64url signature, no padding>
```

- `X-Invoice-Timestamp` — Unix time in seconds, UTC, stamped at delivery time.
- `X-Invoice-Key-Id` — identifies which signing key was used (matches a `kid` from the public key endpoint below).
- `X-Invoice-Signature` — an Ed25519 signature, tagged `v1=`.

Signed message: `timestamp + "." + raw_request_body_bytes` — the timestamp is part of what's signed, so a captured request can't be replayed later with a different body.

**Public keys**: fetch current signing keys from `GET /v1/webhooks/public_key` (no authentication required) — a standard JWKS document:

```json
{ "keys": [ { "kty": "OKP", "crv": "Ed25519", "alg": "EdDSA", "use": "sig", "kid": "…", "x": "<base64url public key>" } ] }
```

Keys can rotate. Cache the JWKS, and if you see an unrecognized `kid`, refetch the JWKS once before rejecting the webhook.

### Payload

The body is the full invoice object (same shape as the invoice API response) as it looked the moment the event was queued:

```json
{
  "id": "0b6c9f0e-2f43-4b8e-9f7a-1c2d3e4f5a6b",
  "merchant_id": "<your merchant id>",
  "user_id": "<payer user id>",
  "client_transaction_id": "order-10345",
  "amount": "25.5",
  "chain_id": 137,
  "ticker": "USDT",
  "platform_fee_percent": "1", "cashback_percent": "0.5",
  "platform_fee_amount": "0.255", "cashback_amount": "0.1275", "net_amount": "25.1175",
  "issued_at": "2026-08-12T10:00:01Z",
  "expires_at": "2026-08-12T10:30:00Z",
  "paid_at": "2026-08-12T10:05:12Z",
  "invoice_status": "completed",
  "invoice_sub_status": "done",
  "p2p_transaction": {
    "id": 42, "sender_id": "<payer user id>", "receiver_id": "<your merchant id>",
    "ticker": "USDT", "chain_id": 137,
    "gross_amount": "25.5", "usd_xrate": "0.999", "service_fee": "0.255",
    "type": "invoice_payment", "created_at": "...", "updated_at": "..."
  },
  "created_at": "...", "updated_at": "..."
}
```

### Your endpoint's contract

- **Respond exactly HTTP `200`** within **5 seconds**. Any other status — including `201` or `204` — is treated as a failure and retried. Acknowledge immediately; do your processing asynchronously.
- **Delivery is at-least-once.** Deduplicate on invoice `id` + `invoice_status` — process each transition once, idempotently.
- **Delivery is FIFO per invoice** — events for the same invoice arrive in order; there's no ordering guarantee across different invoices.
- **Retries** use exponential backoff with jitter over a bounded number of attempts. After the final attempt, a webhook that couldn't be delivered is marked permanently failed — **there is no redelivery mechanism**. This is why reconciliation (below) is mandatory, not optional.

### Verifying signatures

Verify against the **raw request body bytes**, before any JSON parsing, using the public key whose `kid` matches `X-Invoice-Key-Id`.

**Go:**

```go
// keys: map[kid]ed25519.PublicKey built from GET /v1/webhooks/public_key
// (public = base64.RawURLEncoding.DecodeString(jwk.X))
func verifyDreamPayWebhook(r *http.Request, keys map[string]ed25519.PublicKey) (bool, []byte) {
    body, _ := io.ReadAll(r.Body)
    ts := r.Header.Get("X-Invoice-Timestamp")

    scheme, encodedSig, ok := strings.Cut(r.Header.Get("X-Invoice-Signature"), "=")
    if !ok || scheme != "v1" {
        return false, body
    }
    sig, err := base64.RawURLEncoding.DecodeString(encodedSig)
    if err != nil || len(sig) != ed25519.SignatureSize {
        return false, body
    }
    public, known := keys[r.Header.Get("X-Invoice-Key-Id")]
    if !known {
        return false, body // refresh the JWKS once, then retry
    }
    return ed25519.Verify(public, []byte(ts+"."+string(body)), sig), body
}
```

**Node.js / Express:**

```js
const crypto = require("crypto");

// jwks: cached result of GET /v1/webhooks/public_key; refresh on unknown kid
function keyFor(kid) {
  const jwk = jwks.keys.find(k => k.kid === kid);
  return jwk && crypto.createPublicKey({ key: { kty: "OKP", crv: "Ed25519", x: jwk.x }, format: "jwk" });
}

app.post("/webhooks/dreampay", express.raw({ type: "application/json" }), (req, res) => {
  const ts = req.header("X-Invoice-Timestamp") ?? "";
  const [scheme, encoded] = (req.header("X-Invoice-Signature") ?? "").split("=", 2);
  const key = keyFor(req.header("X-Invoice-Key-Id") ?? "");
  const ok = scheme === "v1" && key &&
    crypto.verify(null, Buffer.concat([Buffer.from(ts + "."), req.body]), key, Buffer.from(encoded, "base64url"));
  if (!ok) return res.sendStatus(401);

  if (Math.abs(Date.now() / 1000 - Number(ts)) > 300) return res.sendStatus(401); // freshness window — your policy

  res.sendStatus(200);                     // ack immediately …
  processEvent(JSON.parse(req.body));      // … then process async, idempotently
});
```

DreamPay doesn't enforce a freshness window on its side — pick your own tolerance (a few minutes is a reasonable default; retried webhooks are re-signed with a fresh timestamp each time, so this doesn't cause false rejections).

### Reconciliation

Because a webhook can permanently fail delivery, and because expiry never triggers a webhook at all, **poll as a backstop**:

```
GET /v1/invoices?status=pending
```

For anything still `pending` past its `expires_at`, treat it as unpaid/abandoned. This is the only way to reliably close out invoices you didn't hear about.

## 5. Error handling

| HTTP | `metadata.error` | Meaning |
|---|---|---|
| 400 | `bad request` | Validation failure — see `metadata.message` for detail |
| 400 | `transactions are not allowed in p2p chains` | Invalid `chain_id` |
| 400 | `invoice payment is disabled for this ticker` | Ticker doesn't currently accept invoice payments |
| 400 | `merchant payout is disabled for this ticker` | Ticker doesn't currently allow payouts |
| 400 | `self-transactions are not allowed` | Payout `user_id` is your own merchant account |
| 400 | `amount can't be less or equal to zero` | Non-positive amount |
| 401 | `api key is empty` | Missing `x-api-key` header |
| 403 | `access forbidden` | Unknown, revoked, or expired API key |
| 403 | `user is banned` | Payout recipient is banned |
| 404 | `ticker not found` / `chain not found` | Unknown ticker or chain |
| 404 | `user with this ID doesn't exist` | Payout recipient not found |
| 404 | `invoice not found` | Wrong invoice ID, or it's not yours |
| 409 | `invoice can't be paid due to status mismatch` / `invoice can't be rejected due to status mismatch` | Invoice is no longer in a state that allows this action |
| 422 | `invoice has expired` | A payer tried to pay after `expires_at` |
| 422 | `insufficient balance of ticker` | Not enough balance to cover the operation |
| 500 | `internal server error` | Unexpected server error — safe to retry with backoff; contact support if it persists |

## 6. Security best practices

1. **Treat your API key like a password.** Anyone holding it can create invoices, read your balances, and send payouts from your balance. Store it in a secret manager, never in client-side code.
2. **Always verify webhook signatures** against the raw request body, using the key selected by `X-Invoice-Key-Id` from the published JWKS. Never act on an unverified webhook — knowing your webhook URL must not be enough to mark an order paid.
3. **Enforce a timestamp freshness window** on incoming webhooks (a few minutes is reasonable).
4. **Handle at-least-once delivery idempotently** — key your fulfillment logic on invoice `id` + status, not on "a webhook arrived."
5. **Reconcile by polling.** A webhook can permanently fail delivery with no redelivery — poll for anything you haven't heard about.
6. **Serve your webhook endpoint over HTTPS.**
7. **Send an idempotency key on every payout**, generated and persisted before you call, and reuse it on retries.
8. **Reconcile against `net_amount` / `gross_amount − service_fee`**, not the raw `amount`.
9. **Rotate your API key** if it may have been exposed, using overlapping keys so nothing goes down mid-rotation.
10. **Don't trust client-side amounts.** Verify `amount`, `ticker`, and `chain_id` from the webhook or a direct API call against your own order record before fulfilling.

## 7. Going live checklist

- [ ] Production API key issued and stored in a secret manager.
- [ ] Webhook endpoint is public HTTPS, responds `200` within 5 seconds, verifies signatures, enforces a freshness window, deduplicates by invoice ID, and processes asynchronously.
- [ ] Webhook URL registered with DreamPay and a test event verified end-to-end.
- [ ] Fee configuration confirmed via `GET /v1/merchant/fees`; your accounting reconciles against `net_amount`.
- [ ] Payouts always sent with idempotency keys.
- [ ] A reconciliation job polls `pending` invoices past `expires_at`.
- [ ] `expires_at` is set to a sensible window at invoice creation.
- [ ] Tickers and chains confirmed against `available-invoice-tickers` / `available-payout-tickers` in production.
- [ ] Alerting set up on `403` responses (key issues) and on webhook signature failures.

## Next

**[User Flows](flows.html)** — sequence diagrams for the invoice, payout, and webhook-delivery lifecycles.
