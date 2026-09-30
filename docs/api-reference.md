---
title: API Reference
layout: default
nav_order: 4
---

# API Reference

[Documentation home](index.md)

The local API runs at `http://localhost:8000`. Interactive OpenAPI documentation is available at `/docs` and `/redoc` while the server is running. These examples describe the request and response models in `app/models/schemas.py`.

## On this page

- [Health Check](#health-check)
- [Quotes](#quotes)
- [Transfers](#transfers)
- [Timeline and Ledger](#timeline-and-ledger)
- [Comparison](#comparison)
- [Admin](#admin)
- [Transfer Statuses](#transfer-statuses)
- [Errors](#errors)

Example IDs and timestamps are illustrative. Use the identifiers returned by your server in subsequent requests. The current API has no authentication layer; the admin routes are local demonstration controls.

## Health Check

`GET /health` returns `200 OK` with `{"status": "ok"}`.

```bash
curl http://localhost:8000/health
```

## Quotes

### Create a quote

`POST /quotes` returns `201 Created`.

- `amount` (number, required): positive source amount.
- `source_currency` (string): defaults to `USD`.
- `target_currency` (string): defaults to `INR`.
- `source_country` and `target_country` (strings): default to `US` and `IN`; they do not select another corridor in this MVP.

Only the USD-to-INR currency pair is supported.

```bash
curl -X POST http://localhost:8000/quotes \
  -H "Content-Type: application/json" \
  -d '{"source_currency": "USD", "target_currency": "INR", "amount": 500.00}'
```

Example response with default configuration:

```json
{
  "quote_id": "qt_abc123",
  "source_amount_usd": 500.0,
  "fx_rate_usd_inr": 83.2,
  "platform_fee_usd": 1.5,
  "network_fee_usd": 0.35,
  "fx_spread_usd": 2.0,
  "recipient_amount_inr": 41279.68,
  "expires_at": "2026-09-26T12:05:00Z"
}
```

The quote deducts the platform fee, network fee, and configured FX spread before converting the remainder at the configured USD/INR rate. For this example, `(500 - 1.50 - 0.35 - 2.00) × 83.20 = 41,279.68 INR`.

### Retrieve a quote

`GET /quotes/{quote_id}` returns `200 OK` with the same response fields, or `404` if the quote does not exist.

```bash
curl http://localhost:8000/quotes/qt_abc123
```

## Transfers

### Create a transfer

`POST /transfers` returns `201 Created`.

- `quote_id` (string, required): ID of an unexpired quote.
- `recipient` (object, required): `name` is required; `bank_account_hint` defaults to an empty string.
- `route_preference` (string): defaults to `lowest_cost`. The value `swift` sets `route_type` to `swift`; other values select `stablecoin`. This flag does not add a live SWIFT connector.
- `idempotency_key` (string, optional): an existing key returns the original transfer. The current implementation does not compare the new payload with the original request or return a conflict for a changed payload.

```bash
curl -X POST http://localhost:8000/transfers \
  -H "Content-Type: application/json" \
  -d '{
    "quote_id": "qt_abc123",
    "recipient": {"name": "Raj Patel", "bank_account_hint": "HDFC ****1234"},
    "route_preference": "lowest_cost",
    "idempotency_key": "remit-001"
  }'
```

Example response:

```json
{
  "transfer_id": "tx_xyz789",
  "quote_id": "qt_abc123",
  "source_amount": 500.0,
  "target_amount_estimated": 41279.68,
  "target_amount_final": null,
  "status": "quoted",
  "route_type": "stablecoin",
  "recipient_name": "Raj Patel",
  "created_at": "2026-09-26T12:00:30Z",
  "updated_at": "2026-09-26T12:00:30Z",
  "completed_at": null
}
```

### Retrieve a transfer

`GET /transfers/{transfer_id}` returns `200 OK` with the same fields and the current status, or `404` if absent.

```bash
curl http://localhost:8000/transfers/tx_xyz789
```

## Timeline and Ledger

`GET /transfers/{transfer_id}/timeline` returns an object containing `transfer_id`, `status`, and an `events` array. Each event exposes `event_type`, `payload`, and `created_at`.

```bash
curl http://localhost:8000/transfers/tx_xyz789/timeline
```

The lifecycle emits `transfer.created`, `transfer.funded`, `treasury.usd_to_usdc.completed`, `blockchain.tx_submitted`, `blockchain.tx_confirmed`, `settlement.initiated`, `settlement.completed`, and `transfer.completed`. A failure emits `transfer.failed`.

`GET /transfers/{transfer_id}/ledger` returns an array of entries. Each exposes `id`, `entry_type`, `account_debit`, `account_credit`, `amount`, `currency`, and `created_at`.

```bash
curl http://localhost:8000/transfers/tx_xyz789/ledger
```

Both endpoints return `404` for an unknown transfer. See [Ledger](ledger.md) for accounting behavior.

## Comparison

`GET /transfers/{transfer_id}/comparison` returns modeled fee and latency comparisons, using configured fee inputs and fixed latency assumptions. These are not live bank quotes or measured settlement performance.

```bash
curl http://localhost:8000/transfers/tx_xyz789/comparison
```

```json
{
  "transfer_id": "tx_xyz789",
  "source_amount_usd": 500.0,
  "stablecoin_route": {
    "total_fee_usd": 4.6,
    "estimated_time_seconds": 750,
    "fee_components": {
      "platform_fee": 1.5,
      "network_fee": 0.35,
      "fx_spread": 2.0,
      "payout_partner_fee": 0.75
    }
  },
  "swift_route": {
    "total_fee_usd": 37.0,
    "estimated_time_seconds": 14400,
    "fee_components": {
      "sender_bank_fee": 15.0,
      "intermediary_fee": 8.0,
      "receiver_fee": 5.0,
      "fx_spread": 9.0
    }
  },
  "fee_savings_usd": 32.4,
  "time_savings_seconds": 13650
}
```

The comparison includes a payout-partner fee in the stablecoin total. The quote's recipient amount does not subtract that extra fee. See [Comparison Engine](comparison.md).

## Admin

### Advance one step

`POST /admin/transfers/{transfer_id}/advance` advances one simulated settlement step and returns `transfer_id`, `previous_status` (currently always `null`), `current_status`, and a message.

```bash
curl -X POST http://localhost:8000/admin/transfers/tx_xyz789/advance
```

### Advance to completion

`POST /admin/transfers/{transfer_id}/advance-all` runs until completion, failure, or an unavailable transition. Starting from a new `quoted` transfer produces:

```json
{
  "transfer_id": "tx_xyz789",
  "final_status": "completed",
  "steps": [
    {
      "from": "quoted",
      "to": "funded"
    },
    {
      "from": "funded",
      "to": "usd_to_usdc_complete"
    },
    {
      "from": "usd_to_usdc_complete",
      "to": "onchain_transfer_pending"
    },
    {
      "from": "onchain_transfer_pending",
      "to": "onchain_transfer_confirmed"
    },
    {
      "from": "onchain_transfer_confirmed",
      "to": "usdc_to_inr_pending"
    },
    {
      "from": "usdc_to_inr_pending",
      "to": "settled"
    },
    {
      "from": "settled",
      "to": "completed"
    }
  ]
}
```

```bash
curl -X POST http://localhost:8000/admin/transfers/tx_xyz789/advance-all
```

### Fail an unfinished transfer

`POST /admin/transfers/{transfer_id}/fail` accepts an optional `reason` query parameter (default `manual`). It returns `transfer_id`, `status: "failed"`, and `reason`. Completed or failed transfers cannot be failed again.

```bash
curl -X POST 'http://localhost:8000/admin/transfers/tx_xyz789/fail?reason=demo_failure'
```

Use a separate unfinished transfer for this example. Admin operations return `404` for unknown transfers; invalid single-step advances or failures return `400`.

## Transfer Statuses

The API creates transfers in `quoted`. Normal progression is:

```text
quoted → funded → usd_to_usdc_complete → onchain_transfer_pending
       → onchain_transfer_confirmed → usdc_to_inr_pending → settled → completed
```

`failed` is also terminal. The enum includes `created`, but the creation endpoint does not use it and the advancement helper has no action for it. See [Settlement](settlement.md).

## Errors

- `400`: unsupported currency pair, expired quote, or invalid transition.
- `404`: quote or transfer not found.
- `422`: request-model validation failure, such as a missing recipient or nonpositive amount.

Application errors have a string `detail`; validation errors contain a list of field-level errors in `detail`.

```json
{"detail": "Quote not found"}
```
