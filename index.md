---
layout: default
title: DreamPay Payment Gateway
permalink: /
---

# DreamPay Payment Gateway

DreamPay is a merchant payment gateway built on a custodial digital-wallet platform. It lets your business **accept payments** from DreamPay app users and **send payouts** to them, settled instantly on an internal ledger.

This site is the integration reference for backend engineers connecting a merchant system to the DreamPay API.

## 1. General overview

### What the gateway does

| Flow | Direction | How it starts | What it's for |
|---|---|---|---|
| **Invoice payment** | Payer → Merchant | You create an invoice via the API; the payer completes it in the DreamPay app | Charging a customer an exact amount for an order |
| **Merchant payout** | Merchant → Payer | You call the payout endpoint | Any payment you initiate to a user — e.g. a refund or disbursement |

Both flows move value between **ledger balances** — internal balances keyed by `(ticker, chain_id)`, e.g. `(USDT, Polygon)`. Movement is instant once the API call succeeds; there is no blockchain confirmation wait on the merchant side.

### Who this is for

You're a merchant (or a developer working on behalf of one) who wants to:
- Charge DreamPay app users for goods or services (invoices), and/or
- Send payments to DreamPay app users (payouts),
- Receive real-time notifications (webhooks) when a payment completes, and
- Reconcile your own records against the DreamPay API.

### Environments

| Environment | Base URL | API docs (Swagger) |
|---|---|---|
| **Production** | `https://api.smartcardpro.com` | [api.smartcardpro.com/swagger/index.html](https://api.smartcardpro.com/swagger/index.html) |
| **Sandbox** | `https://api-sandbox.dreampay.cards` | [api-sandbox.dreampay.cards/swagger/index.html](https://api-sandbox.dreampay.cards/swagger/index.html) |

All routes are versioned under `/v1`. The Swagger spec does not include a host, so prepend the base URL for the environment you're calling — e.g. `POST https://api.smartcardpro.com/v1/invoices`.

Use sandbox for integration testing, then switch to production credentials to go live. Confirm with your DreamPay account manager which chains and tickers are enabled in sandbox and how to fund a test account — these may differ from production.

### Supported assets

Ledger balances are identified by a `(ticker, chain_id)` pair. In **production**, DreamPay supports stablecoins and its native token across Ethereum (`chain_id 1`), Polygon (`chain_id 137`) and Tron (`chain_id 728126428`). Sandbox may enable a different set of chains and tickers (for example testnets) — confirm with your account manager before hard-coding chain IDs against sandbox.

The exact list of tickers and chains you can invoice or pay out with is dynamic in every environment — always discover it at call time via `GET /v1/merchant/available-invoice-tickers` and `GET /v1/merchant/available-payout-tickers` (see the [Integration Guide](integration.html)) rather than hard-coding it.

## Get started

Merchant onboarding (profile, webhook URL, and your API key) is provisioned by the DreamPay team — there is no self-service signup. Contact your DreamPay account manager (`<merchant-onboarding contact — to be filled in>`) to get set up, then continue with the [Integration Guide](integration.html).

## Next

- **[Integration Guide](integration.html)** — authentication, request/response conventions, the invoice and payout API, webhook setup and verification, error handling.
- **[User Flows](flows.html)** — sequence diagrams for the invoice, payout, and webhook-delivery lifecycles.
