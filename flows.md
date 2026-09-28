---
title: User Flows
---

# User Flows

[Overview](index.html) · [Integration Guide](integration.html) · **[User Flows](flows.html)**

Sequence diagrams for the flows described in the [Integration Guide](integration.html). "Merchant backend" is your server, calling the DreamPay API with your `x-api-key`. "Payer" is the DreamPay app user completing or rejecting a payment.

## Invoice payment — happy path

<div class="mermaid">
sequenceDiagram
    participant M as Merchant backend
    participant DP as DreamPay API
    participant U as Payer (DreamPay app)

    M->>DP: POST /v1/invoices (x-api-key, idempotency_key)
    DP-->>M: 201 invoice { id, invoice_status: pending }
    M->>U: Deliver invoice id (deeplink / QR) via your own channel
    U->>DP: POST /v1/account/invoices/{id}/pay
    DP-->>U: 200 paid
    DP-->>M: Webhook POST invoice.completed (Ed25519-signed)
    M->>M: Verify signature, process order, respond 200
    M->>DP: GET /v1/invoices/{id} (optional reconciliation check)
</div>

## Invoice rejected or cancelled

An unclaimed invoice can be rejected by the payer, or cancelled by you (the merchant) — but not both; once a payer touches it, you can no longer cancel it.

<div class="mermaid">
sequenceDiagram
    participant M as Merchant backend
    participant DP as DreamPay API
    participant U as Payer (DreamPay app)

    alt Payer rejects
        U->>DP: POST /v1/account/invoices/{id}/reject
    else Merchant cancels (only while unclaimed)
        M->>DP: POST /v1/invoices/{id}/reject
        DP-->>M: 200 invoice now rejected
        Note over M,DP: 409 if a payer has already claimed it
    end
    DP-->>M: Webhook POST invoice.rejected
    M->>M: Verify signature, release/cancel order
</div>

## Invoice expiry — no webhook, poll instead

Expiry is enforced only at the moment a payer attempts to pay — it never changes status on its own and never fires a webhook. Reconciliation by polling is required to catch this case.

<div class="mermaid">
sequenceDiagram
    participant M as Merchant backend
    participant DP as DreamPay API
    participant U as Payer (DreamPay app)

    Note over M,U: expires_at passes — no status change,<br/>no webhook
    U->>DP: POST /v1/account/invoices/{id}/pay (too late)
    DP-->>U: 422 invoice has expired

    loop Merchant reconciliation (e.g. every few minutes)
        M->>DP: GET /v1/invoices?status=pending
        DP-->>M: pending invoices
        M->>M: If expires_at has passed, treat as unpaid
    end
</div>

## Merchant payout

<div class="mermaid">
sequenceDiagram
    participant M as Merchant backend
    participant DP as DreamPay API
    participant U as Recipient (DreamPay app)

    M->>DP: GET /v1/merchant/users/{user_id} (optional pre-check)
    DP-->>M: { exists: true }
    M->>DP: POST /v1/merchant/payouts (idempotency_key)
    DP-->>M: 201 payout { gross_amount, service_fee }
    DP-->>U: Balance credited, push notification
</div>

## Webhook delivery and retries

<div class="mermaid">
sequenceDiagram
    participant DP as DreamPay
    participant M as Merchant webhook endpoint

    DP->>M: POST webhook_url (attempt 1, Ed25519-signed)
    alt Merchant responds 200 within 5s
        M-->>DP: 200 OK
        Note over DP: Delivered
    else Timeout or non-200
        DP->>DP: Schedule retry (exponential backoff)
        DP->>M: POST webhook_url (attempt N)
    end
    Note over DP,M: After the final attempt with no 200,<br/>the webhook is marked permanently not delivered.<br/>Reconciliation (polling) is the only recovery.
</div>

## Back

[Integration Guide](integration.html) · [Overview](index.html)

<script src="https://cdn.jsdelivr.net/npm/mermaid@10/dist/mermaid.min.js"></script>
<script>mermaid.initialize({ startOnLoad: true, theme: 'neutral' });</script>
