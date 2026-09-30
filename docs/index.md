---
title: Home
layout: default
nav_order: 1
---

# Cross-Border Payments Engine

A remittance demonstration for USD-to-INR transfers, with FX quotes, a settlement state machine, double-entry ledger accounting, and modeled stablecoin-versus-SWIFT comparisons.

[Get Started](getting-started.md) · [API Reference](api-reference.md) · [Repository README](https://github.com/vaibhavkapur/Cross-Border-Payments-Engine/blob/master/README.md)

## Documentation

- [Getting Started](getting-started.md)
- [Architecture](architecture.md)
- [API Reference](api-reference.md)
- [Configuration](configuration.md)
- [Database Schema](database.md)
- [Testing](testing.md)
- [Deployment](deployment.md)
- [Settlement State Machine](settlement.md)
- [FX & Quote Engine](fx-engine.md)
- [Ledger System](ledger.md)
- [Blockchain Integration](blockchain.md)
- [Comparison Engine](comparison.md)
- [Workers](workers.md)

## Overview

The engine demonstrates the lifecycle of a remittance through FX quoting, fee calculation, settlement transitions, and ledger posting. Its stablecoin and SWIFT comparison uses configured assumptions rather than live bank quotes.

Fee and latency figures in the comparison guides are modeled estimates. They are not measured settlement times or commitments from a payment provider.

## Key Features

- **FX Quote Engine** — Configured USD/INR quotes with transparent fee breakdown (platform, network, FX spread)
- **Settlement State Machine** — 10-state lifecycle with enforced valid transitions and event sourcing
- **Double-Entry Ledger** — Full accounting trail for every value movement across the transfer lifecycle
- **Blockchain Integration** — Simulated USDC transfers on Base Sepolia with transaction tracking
- **SWIFT Benchmarking** — Side-by-side fee and latency comparison against incumbent rails
- **Idempotent Transfers** — Safe retries via idempotency keys on transfer creation
- **Admin Controls** — Manual state advancement and failure injection for testing
- **Event Timeline** — Chronological audit trail of every settlement step

## Architecture at a Glance

```
┌─────────────┐     ┌──────────────────────────────────────────────┐
│   Client     │     │          Cross-Border Payments Engine         │
│  (REST API)  │────▶│                                              │
└─────────────┘     │  ┌──────────┐  ┌────────────┐  ┌──────────┐ │
                    │  │  Quotes  │  │  Transfers  │  │  Admin   │ │
                    │  │   API    │  │    API      │  │   API    │ │
                    │  └────┬─────┘  └─────┬──────┘  └────┬─────┘ │
                    │       │              │               │       │
                    │  ┌────▼──────────────▼───────────────▼─────┐ │
                    │  │           Service Layer                  │ │
                    │  │  ┌──────────┐ ┌─────────┐ ┌──────────┐ │ │
                    │  │  │ FX Engine│ │Settlement│ │Comparison│ │ │
                    │  │  └──────────┘ │ Service  │ │  Engine  │ │ │
                    │  │               └─────────┘ └──────────┘ │ │
                    │  └────────────────────┬──────────────────┘ │
                    │                       │                     │
                    │  ┌────────────────────▼──────────────────┐ │
                    │  │          Data & Integration Layer       │ │
                    │  │  ┌──────┐ ┌──────────┐ ┌───────────┐ │ │
                    │  │  │Ledger│ │Blockchain │ │  Database  │ │ │
                    │  │  │      │ │ Simulator │ │(PostgreSQL)│ │ │
                    │  │  └──────┘ └──────────┘ └───────────┘ │ │
                    │  └──────────────────────────────────────┘ │
                    └──────────────────────────────────────────────┘
```

## Tech Stack and Scope

Python / FastAPI / SQLAlchemy / Celery, with SQLite or PostgreSQL and Redis. FX rates and comparison inputs are configured values; USDC transfers and local payouts are simulated.

| Component | Technology |
|:----------|:-----------|
| **Backend Framework** | FastAPI 0.115 + Uvicorn |
| **Database** | PostgreSQL 16 / SQLite (dev) |
| **ORM** | SQLAlchemy 2.0 (async) |
| **Migrations** | Alembic 1.14 |
| **Task Queue** | Celery 5.4 + Redis 7 |
| **Validation** | Pydantic 2.10 |
| **Blockchain** | Base Sepolia (simulated USDC) |
| **Containerization** | Docker + Docker Compose |
| **Language** | Python 3.12 |

## Project Structure

```
Cross-Border-Payments-Engine/
├── app/
│   ├── api/                    # FastAPI route handlers
│   │   ├── admin.py            # Settlement state controls
│   │   ├── comparison.py       # Stablecoin vs SWIFT comparison
│   │   ├── quotes.py           # FX quote generation
│   │   └── transfers.py        # Transfer CRUD & timeline
│   ├── blockchain/
│   │   └── simulator.py        # Simulated USDC transfers
│   ├── comparison/
│   │   └── engine.py           # Fee & latency benchmarking
│   ├── fx/
│   │   └── engine.py           # FX rate & quote calculation
│   ├── ledger/
│   │   └── service.py          # Double-entry ledger posting
│   ├── models/
│   │   ├── enums.py            # State machine & transitions
│   │   ├── schemas.py          # Pydantic request/response models
│   │   └── tables.py           # SQLAlchemy ORM tables
│   ├── services/
│   │   └── settlement.py       # Settlement orchestration
│   ├── workers/
│   │   └── celery_app.py       # Background task definitions
│   ├── config.py               # Application settings
│   ├── database.py             # Async DB engine & sessions
│   └── main.py                 # FastAPI app entrypoint
├── migrations/                  # Alembic migration scripts
├── scripts/
│   └── demo.sh                 # End-to-end demo script
├── docker-compose.yml           # Local dev environment
├── Dockerfile                   # Container image
├── requirements.txt             # Python dependencies
└── alembic.ini                  # Migration configuration
```

## Related projects

These are independent companion repositories. The links describe related work, not implemented runtime integrations:

- [Agent Authorization Wallet + Merchant Trust Gateway](https://github.com/vaibhavkapur/Agent-Authorization-Wallet-Merchant-Trust-Gateway): purchase authorization, merchant verification, and execution evidence.
- [Agent Services Marketplace](https://github.com/vaibhavkapur/Agent-Services-Marketplace): service discovery, quotes, and agent purchase workflows.
- [Agentic Commerce Protocol Test Lab](https://github.com/vaibhavkapur/Agentic-Commerce-Protocol-Test-Lab): protocol fixtures, scenarios, and conformance checks.
- [Autonomous Price Watch Buyer](https://github.com/vaibhavkapur/Autonomous-Price-Watch-Buyer): price monitoring and bounded purchase decisions.
- [Cross-Merchant Procurement Agent](https://github.com/vaibhavkapur/Cross-Merchant-Procurement-Agent): merchant comparison and procurement planning.
