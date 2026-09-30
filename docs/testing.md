---
title: Testing
layout: default
nav_order: 20
---

# Testing

[Documentation home](index.md)

## Local smoke test

Follow [Getting Started](getting-started.md) to start the API, then run the checked-in demonstration from the repository root:

```bash
bash scripts/demo.sh http://localhost:8000
```

The script creates a quote and transfer, advances the simulated settlement pipeline, and prints the timeline, ledger, comparison, and final transfer. Check that the final status is `completed` and the returned identifiers match throughout the flow.

The repository currently has no automated test suite. The demo is a manual smoke check, not an assertion-based regression test. Settlement and comparison results use simulation and configured estimates.

## Failure and retry checks

- Reuse a transfer's idempotency key and check that the original `transfer_id` is returned.
- Submit a nonpositive quote amount and expect request validation to reject it.
- On a separate unfinished transfer, use the failure endpoint in [API Reference](api-reference.md#admin) and inspect its timeline.

See [Settlement](settlement.md) and [Ledger](ledger.md) for the lifecycle and accounting details.
