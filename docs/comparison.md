---
title: Comparison Engine
layout: default
nav_order: 9
---

# Comparison Engine

[Documentation home](index.md)

Side-by-side benchmarking of stablecoin transfers against SWIFT for fees, latency, and recipient value.

## On this page

- [Overview](#overview)
- [Stablecoin Route Breakdown](#stablecoin-route-breakdown)
- [SWIFT Route Breakdown](#swift-route-breakdown)
- [Savings Calculation](#savings-calculation)
- [Comparison Response Schema](#comparison-response-schema)
- [Benchmark Storage](#benchmark-storage)
- [Configuration](#configuration)

---

## Overview

The comparison engine (`app/comparison/engine.py`) generates modeled comparisons from configured fees and fixed latency assumptions. For every completed transfer, it calculates detailed fee and latency breakdowns for both routes and computes the savings.

## Stablecoin Route Breakdown

### Fee Structure

| Component | Amount | Description |
|:----------|:-------|:------------|
| Platform Fee | $1.50 | Service fee |
| Network Fee | $0.35 | Blockchain gas costs |
| FX Spread | $2.00 | 0.4% of $500 |
| Payout Partner Fee | $0.75 | Off-ramp partner cost |
| **Total** | **$4.60** | |

### Latency Breakdown

| Step | Duration | Description |
|:-----|:---------|:------------|
| Funding | 120s (2 min) | Receive sender's USD payment |
| USD → USDC | 30s | Treasury conversion |
| On-chain Transfer | 90s (1.5 min) | USDC transaction broadcast |
| Confirmation | 30s | 12 block confirmations |
| Off-ramp Prep | 180s (3 min) | USDC → INR conversion |
| Local Payout | 300s (5 min) | INR delivery to bank |
| **Total** | **750s (12.5 min)** | |

## SWIFT Route Breakdown

### Fee Structure (Estimated)

| Component | Amount | Description |
|:----------|:-------|:------------|
| Sender Bank Fee | $15.00 | Originating bank wire fee |
| Intermediary Fee | $8.00 | Correspondent bank fee |
| Receiver Fee | $5.00 | Beneficiary bank fee |
| FX Spread | $9.00 | 1.8% retail markup on $500 |
| **Total** | **$37.00** | |

### Latency (Estimated)

| Route | Time |
|:------|:-----|
| SWIFT (total) | ~4 hours (14,400 seconds) |

Four hours is a fixed modeling assumption in this implementation; it is not a live estimate from a bank or settlement provider.

## Savings Calculation

For a USD 500 → INR transfer:

```
Fee Savings
  SWIFT fees:       $37.00
  Stablecoin fees:   $4.60
  ────────────────────────
  Saved:            $32.40  (87.6%)

Time Savings
  SWIFT time:       14,400s  (4 hours)
  Stablecoin time:     750s  (12.5 minutes)
  ────────────────────────
  Saved:            13,650s  (94.8%)

```

The quote calculates the stablecoin recipient amount separately. Its deduction excludes the payout-partner fee included in this comparison, and this endpoint does not calculate a SWIFT recipient amount.

## Comparison Response Schema

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

## Benchmark Storage

Comparison results are persisted in the `benchmarks` table for analytics:

| Field | Type | Description |
|:------|:-----|:------------|
| `id` | string | Benchmark record identifier |
| `transfer_id` | string | Associated transfer |
| `stablecoin_total_fee` | decimal | Total stablecoin route fees (USD) |
| `stablecoin_total_time_sec` | integer | Total stablecoin settlement time |
| `swift_estimated_fee` | decimal | Estimated SWIFT fees (USD) |
| `swift_estimated_time_sec` | integer | Estimated SWIFT settlement time |
| `created_at` | datetime | When benchmark was recorded |

## Configuration

SWIFT comparison estimates are configurable:

| Variable | Default | Description |
|:---------|:--------|:------------|
| `SWIFT_SENDER_BANK_FEE` | `15.00` | Originating bank fee |
| `SWIFT_INTERMEDIARY_FEE` | `8.00` | Correspondent bank fee |
| `SWIFT_RECEIVER_FEE` | `5.00` | Beneficiary bank fee |
| `SWIFT_FX_SPREAD_PCT` | `0.018` | SWIFT retail FX spread (1.8%) |
