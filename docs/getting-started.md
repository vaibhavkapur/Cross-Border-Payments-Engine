---
title: Getting Started
layout: default
nav_order: 2
---

# Getting Started

[Documentation home](index.md)

Set up the Cross-Border Payments Engine locally and execute your first USD→INR transfer in minutes.

## On this page

- [Prerequisites](#prerequisites)
- [Clone the Repository](#clone-the-repository)
- [Quick Start (SQLite)](#quick-start-sqlite)
- [Docker Compose Setup](#docker-compose-setup)
- [Environment Configuration](#environment-configuration)
- [Run Migrations](#run-migrations)
- [Make Your First Transfer](#make-your-first-transfer)
- [Run the Demo Script](#run-the-demo-script)
- [What's Next](#whats-next)

---

## Prerequisites

| Requirement | Version |
|:------------|:--------|
| Python | 3.12+ |
| Docker & Docker Compose | Latest |
| PostgreSQL | 16+ (or use SQLite for quick start) |
| Redis | 7+ (for Celery workers) |
| Git | 2.x+ |

## Clone the Repository

```bash
git clone https://github.com/vaibhavkapur/Cross-Border-Payments-Engine.git
cd Cross-Border-Payments-Engine
```

## Quick Start (SQLite)

The fastest way to get running — uses SQLite and skips Docker dependencies.

```bash
# Create virtual environment
python -m venv venv
source venv/bin/activate

# Install dependencies
pip install -r requirements.txt
# SQLite driver for the default local database
pip install aiosqlite

# Run database migrations
alembic upgrade head

# Start the API server
uvicorn app.main:app --reload --port 8000
```

Verify the server is running:

```bash
curl http://localhost:8000/health
```

```json
{
  "status": "ok"
}
```

## Docker Compose Setup

For a full production-like environment with PostgreSQL and Redis:

```bash
# Start all services
docker compose up -d

# Verify services are running
docker compose ps
```

This starts four services:

| Service | Port | Description |
|:--------|:-----|:------------|
| `db` | 5432 | PostgreSQL 16 database |
| `redis` | 6379 | Redis 7 message broker |
| `api` | 8000 | FastAPI application (hot reload) |
| `worker` | — | Celery background worker |

## Environment Configuration

Create a `.env` file in the project root. The defaults work out of the box for Docker Compose:

```bash
# Database
DATABASE_URL=postgresql+asyncpg://payments:payments@localhost:5432/cross_border

# Redis (for Celery)
REDIS_URL=redis://localhost:6379/0

# FX Configuration (optional — defaults shown)
USD_INR_MID_RATE=83.20
STABLECOIN_FX_SPREAD_PCT=0.004
PLATFORM_FEE=1.50
NETWORK_FEE=0.35
QUOTE_TTL_SECONDS=300
```

## Run Migrations

```bash
# With Docker Compose
docker compose exec api alembic upgrade head

# Without Docker
alembic upgrade head
```

## Make Your First Transfer

### Step 1: Create a Quote

Request an FX quote for a USD 500 → INR transfer:

```bash
curl -s -X POST http://localhost:8000/quotes \
  -H "Content-Type: application/json" \
  -d '{
    "source_currency": "USD",
    "target_currency": "INR",
    "amount": 500.00
  }' | python -m json.tool
```

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

The quote is valid for 5 minutes by default; example timestamps are illustrative. Fee breakdown:
- **Platform fee**: $1.50
- **Network fee**: $0.35
- **FX spread** (0.4%): $2.00
- **Net amount converted**: $496.15 × 83.20 = **₹41,279.68**

### Step 2: Create a Transfer

Use the returned `quote_id` to initiate the transfer. Replace the illustrative `qt_abc123` below and use the returned `transfer_id` in later steps:

```bash
curl -s -X POST http://localhost:8000/transfers \
  -H "Content-Type: application/json" \
  -d '{
    "quote_id": "qt_abc123",
    "recipient": {"name": "Raj Patel", "bank_account_hint": "HDFC ****1234"},
    "idempotency_key": "txn-001-unique"
  }' | python -m json.tool
```

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

### Step 3: Execute the Full Pipeline

Use the admin endpoint to advance the transfer through all settlement states:

```bash
curl -s -X POST http://localhost:8000/admin/transfers/tx_xyz789/advance-all \
  | python -m json.tool
```

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

### Step 4: View the Timeline

Check the complete event history:

```bash
curl -s http://localhost:8000/transfers/tx_xyz789/timeline \
  | python -m json.tool
```

### Step 5: Compare Against SWIFT

See the fee and latency savings:

```bash
curl -s http://localhost:8000/transfers/tx_xyz789/comparison \
  | python -m json.tool
```

## Run the Demo Script

A complete end-to-end demo is available:

```bash
chmod +x scripts/demo.sh
./scripts/demo.sh
```

This executes a full USD 500 → INR transfer, advancing through every settlement state and displaying the comparison results.

## What's Next

- [Architecture](architecture.md) — Understand the system design and payment lifecycle
- [API Reference](api-reference.md) — Full endpoint documentation
- [Settlement State Machine](settlement.md) — Deep dive into the 10-state lifecycle
- [Ledger System](ledger.md) — How double-entry accounting works
