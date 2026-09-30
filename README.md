# Cross-Border Payments Engine

A remittance demonstration for USD-to-INR transfers, with FX quotes, a settlement state machine, double-entry ledger accounting, and modeled stablecoin-versus-SWIFT comparisons.

> **[Read the full documentation](docs/index.md)**

Python / FastAPI / SQLAlchemy / Celery, with SQLite or PostgreSQL and Redis. FX rates and comparison inputs are configured values; USDC transfers and local payouts are simulated.

## Getting Started

See the [Getting Started guide](docs/getting-started.md) for prerequisites and configuration.

```bash
# Start Postgres & Redis
docker compose up -d db redis

# Install dependencies
pip install -r requirements.txt
# SQLite driver for the default local database
pip install aiosqlite

# Run migrations
alembic upgrade head

# Start the API server
uvicorn app.main:app --reload --port 8000

# Start the Celery worker (separate terminal)
celery -A app.workers.celery_app worker --loglevel=info
```

## Quick Example

Replace `qt_...` and `{transfer_id}` with the identifiers returned by the API. Settlement is simulated.

```bash
# Create a quote
curl -X POST http://localhost:8000/quotes \
  -H "Content-Type: application/json" \
  -d '{
    "source_currency": "USD",
    "target_currency": "INR",
    "amount": 500.00
  }'

# Create a transfer from the quote
curl -X POST http://localhost:8000/transfers \
  -H "Content-Type: application/json" \
  -d '{
    "quote_id": "qt_...",
    "recipient": {"name": "Raj Patel", "bank_account_hint": "HDFC ****1234"},
    "idempotency_key": "remit-001"
  }'

# Advance through the full settlement pipeline
curl -X POST http://localhost:8000/admin/transfers/{transfer_id}/advance-all

# Compare stablecoin vs SWIFT fees
curl http://localhost:8000/transfers/{transfer_id}/comparison
```
