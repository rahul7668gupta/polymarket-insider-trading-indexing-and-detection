# Polymarket Insider Trading Detection

Architecture and implementation plan for a real-time system that detects potential insider trading on [Polymarket](https://polymarket.com) by monitoring on-chain `OrderFilled` events on Polygon.

---

## What this is

A design-stage repository containing the full system architecture, detection algorithm, and operational runbooks for an event-driven Go service. No code exists yet — the files here are the plan that guides implementation.

The system:

- Listens to `OrderFilled` events from Polymarket's CTF Exchange contracts on Polygon
- Scores each trader (maker and taker) against 5 signals: entry timing, market count, trade size, wallet age, and position concentration
- Sends Telegram alerts for wallets that exceed a suspicion threshold
- Maintains a full historical backfill of known insider wallets (will also be used for backtesting and model tuning)

---

## Files


| File                                              | What it is                                                                     |
| ------------------------------------------------- | ------------------------------------------------------------------------------ |
| `polymarket_insider_trading_detection_plan.md`    | Architecture plan — all sections, design decisions, and rationale. Start here. |
| `polymarket_insider_trading_detection_diagram.md` | 8 Mermaid flow diagrams — every subsystem visualised end-to-end.               |


---

## Reading path

```
polymarket_insider_trading_detection_plan.md   ← start here
    Section 1:  Event Listener (Polygon WebSocket, 10-block depth)
    Section 2:  USDC.e Deposit Lookup (The Graph subgraph)
    Section 3:  Kafka Topics & Schema
    Section 4:  Postgres Schema
    Section 5:  Detection Algorithm (5 signals, dual maker/taker scoring)
    Section 6:  Parameter Tuning & Validation
    Section 7:  Telegram Alerting
    Section 8:  Redis Cache Layer (4 key types)
    Section 9:  Historical Backfill Service
    Section 10: Telemetry (Prometheus + Grafana)
    Section 11: DevOps (Docker Compose, CI/CD, Alertmanager)

polymarket_insider_trading_detection_diagram.md   ← visualise the flows
    Diagram 1: End-to-end system architecture
    Diagram 2: Live event path (block → Kafka → consumer)
    Diagram 3: Detection algorithm (dual maker/taker scoring)
    Diagram 4: Detection steps 1–5 in detail
    Diagram 5: Redis cache read-through pattern (all 4 key types)
    Diagram 6: Historical backfill pipeline
    Diagram 7: Telemetry and metrics collection
    Diagram 8: Alert escalation chain
```

---

## Key design decisions at a glance


| Decision                    | Choice                                  | Why                                                       |
| --------------------------- | --------------------------------------- | --------------------------------------------------------- |
| Blockchain indexing         | WebSocket + 10-block confirmation depth | No reorg handling needed; ~20s latency acceptable         |
| First-deposit history       | The Graph subgraph (not self-indexed)   | 3+ years of history available on day 1, zero extra infra  |
| Message bus                 | Kafka (3 topics)                        | Decouples listener from consumer; enables backfill replay |
| Both maker and taker scored | Yes, independently                      | Maker-side insiders exist; costs one extra Redis lookup   |
| Cache                       | Redis (4 key types, 1ms hot path)       | Shared across consumer replicas; survives restarts        |
| Alert threshold             | 0.60 flag / 0.80 auto-alert             | Tuned via grid search on 7 known insider wallets          |


