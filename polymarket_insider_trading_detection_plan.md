# Polymarket Insider Trading Detection — Plan

## Overview

An event-driven system that monitors Polymarket's on-chain `OrderFilled` events in real time, scores each trading wallet across five behavioural signals, and fires Telegram alerts when a wallet's composite score exceeds an insider threshold. The same pipeline runs over historical data (backfill) to surface past insider activity.

---

## Architecture

```
  Polygon (CTF Exchange + Neg Risk CTF)
       │  WebSocket — 10-block confirmation depth
       v
  Event Listener  →  Kafka [order-filled]
                             │
                             v
                      Consumer Service
                      ┌─────────────────────────────────┐
                      │ Indexer Worker  → PostgreSQL     │
                      │ Detection Channel → score + route│
                      │ Retry Consumer  → re-runs        │
                      └─────────────────────────────────┘
                             │ cache miss (Step 1)
                             v
                      The Graph — USDC.e Subgraph
                      (first deposit lookup, cached 24h in Redis)
```

**Three Kafka topics:**

| Topic                   | Purpose                                          |
| ----------------------- | ------------------------------------------------ |
| `order-filled`          | Live OrderFilled events → indexer + detection    |
| `order-filled-backfill` | Historical events → backfill consumer            |
| `detection-retry`       | Stale detections (race condition) → retry worker |

---

## Components

### 1. Event Listener

Subscribes to the Polygon WebSocket and captures `OrderFilled` events from both CTF Exchange contracts. Applies a **10-block confirmation depth** — only processes logs from blocks at least 10 behind the chain tip, eliminating all reorg handling. On each confirmed block, fetches logs, decodes events, and publishes to Kafka. Tracks the last confirmed block in Postgres so restarts resume without gaps.

The 10-block depth adds ~20 seconds of latency, which is acceptable since insider markets resolve over hours or days.

---

### 2. USDC.e Deposit Lookup (The Graph Subgraph)

Rather than running a parallel Kafka pipeline to index USDC.e Transfer events, the system queries **The Graph's USDC.e subgraph** on demand. The subgraph has indexed all transfers from genesis — there is no cold-start gap.

**Lookup path:** Redis GET (1ms hit) → GraphQL query to The Graph (50–150ms miss) → SET Redis with 24h TTL.

| Dimension           | Self-indexed pipeline      | The Graph subgraph                    |
| ------------------- | -------------------------- | ------------------------------------- |
| Historical coverage | Only from deployment date  | Complete from genesis                 |
| Extra infrastructure| Kafka topic + DB table     | None                                  |
| Day-1 correctness   | No                         | Yes — 3+ years already indexed        |
| Query latency       | 5–20ms (DB)                | 50–150ms first fetch, 1ms after cache |

**Subgraph sync-lag monitoring:** A background goroutine polls `_meta` every 30 seconds and exposes `subgraph_last_indexed_block`. If the subgraph falls more than 50 blocks behind the confirmed chain tip, cache misses may return incomplete data — wallet-age scores inflate, causing false positives. Grafana alerts fire at 50 blocks (warning) and 200 blocks (critical).

---

### 3. Consumer Service

A single Go binary configured via environment variables, used for both live and backfill paths. Runs three goroutine pools:

- **3a. Indexer Worker** — writes trades, upserts wallets, fetches and caches market metadata from the Polymarket API.
- **3b. Detection Channel Listener** — receives events via a Go channel, runs the full detection algorithm (Section 5), routes outcomes to Telegram / DB update / retry topic.
- **3c. Retry Consumer** — reads the `detection-retry` topic, waits briefly, then re-runs detection once the trade is confirmed in Postgres.

---

### 4. PostgreSQL (Data Store)

**Tables:** `trades`, `wallets`, `markets`, `alerts`, `indexer_state`, `backfill_progress`.

Key design points:
- `trades` has a `UNIQUE(tx_hash, log_index)` constraint — all writes are idempotent (`ON CONFLICT DO NOTHING`), safe for backfill/live overlap.
- `wallets` stores the cached insider score, flag status, and aggregate trade stats — updated after each detection run.
- `backfill_progress` tracks per-wallet chunk progress so backfill is resumable after a crash.
- First-deposit data is **never stored in Postgres** — Redis and The Graph subgraph are the sole source.

---

### 5. Detection Algorithm

Every `OrderFilled` event is scored **twice** — once for the maker address, once for the taker address. Both are evaluated independently. The taker (actively filling an order) is the more likely insider, but maker-side insiders exist too (large limit orders placed before a resolution event). The two scoring runs share cached market metadata; only the wallet-specific lookups differ.

**Step 1 — First USDC.e Deposit (run once per address)**
Redis GET → cache hit uses cached timestamp. Cache miss queries The Graph subgraph and writes result to Redis (24h TTL).

**Step 2 — Market Metadata**
Redis GET for token→market mapping (immutable, no TTL) → Postgres on miss → Polymarket Gamma API if not in DB → write to Redis (1h TTL).

**Step 3 — Wallet Profile**
Redis GET for wallet profile (5-minute TTL) → Postgres on miss. Computes: total trades, distinct markets, total volume, concentration ratio. 5-minute TTL keeps the profile fresh while the indexer writes concurrently; short enough to stay accurate, long enough to collapse repeated detections for the same wallet within a backfill chunk.

**Step 4 — Score 5 Signals**

All signals return a value in [0.0, 1.0] where 1.0 = maximum suspicion.

| # | Signal       | Weight | What it measures                                               |
|---|--------------|--------|----------------------------------------------------------------|
| 1 | Entry timing | 0.25   | How close to market resolution the trade was placed            |
| 2 | Market count | 0.20   | Few distinct markets = concentrated, suspicious activity       |
| 3 | Trade size   | 0.20   | Large absolute or relative position in a single market         |
| 4 | Wallet age   | 0.15   | Short gap between first deposit and first trade                |
| 5 | Concentration| 0.20   | Fraction of total wallet volume in a single market             |

**Step 5 — Aggregate Score & Route**

Composite score = weighted sum of the five signals. Weights are config-driven (Viper) and set by the tuning script — no recompile needed.

| Score      | Decision                                 |
| ---------- | ---------------------------------------- |
| ≥ 0.80     | HIGH — Telegram alert + flag in DB       |
| ≥ 0.60     | MEDIUM — Telegram alert + flag in DB     |
| < 0.60     | LOW — update wallet as not-insider in DB |

---

### 6. Parameter Tuning & Validation

The five signal weights and thresholds are found empirically via a **grid search** over the labeled dataset of known insider wallets.

**Ground truth dataset:**

| Wallet                                       | Market          | Alias         |
| -------------------------------------------- | --------------- | ------------- |
| `0xee50a31c3f5a7c77824b12a941a54388a2827ed6` | Google d4vd     | alpha raccoon |
| `0x6baf05d193692bb208d616709e27442c910a94c5` | Maduro out      | SBet365       |
| `0x31a56e9e690c621ed21de08cb559e9524cdb8ed9` | Maduro out      | unnamed       |
| `0x0afc7ce56285bde1fbe3a75efaffdfc86d6530b2` | Israel Iran     | ricosuave     |
| `0x7f1329ade2ec162c6f8791dad99125e0dc49801c` | Trump pardon CZ | gj1           |
| `0x976685b6e867a0400085b1273309e84cd0fc627c` | MicroStrategy   | fromagi       |
| `0x55ea982cebff271722419595e0659ef297b48d7c` | DraftKings      | flaccidwillie |

Plus ~100 normal wallets (sampled from the Polymarket CLOB API) as negative examples. 80 are used for tuning, 20 held out for final FP-rate validation.

**`scripts/tune_params.go`** — enumerates all weight combinations where each weight ∈ {0.05…0.95} in steps of 0.05 and the sum equals 1.0 (~2,380 combinations). For each combination × 7 thresholds (0.50–0.80), computes TP/FP/FN and F1 score. Writes the best config to `config.yaml`. Runtime < 1 second.

**`scripts/validate.go`** — runs the chosen config against both tune and hold-out sets, prints a confusion matrix, per-wallet scores, and a score histogram for negative wallets. Re-run whenever new insider wallets are added to `backfill_wallets.yaml`.

---

### 7. Telegram Alerting

When a wallet scores ≥ 0.60, a structured alert is sent to a configured Telegram channel via the Bot API. The alert includes: composite score, per-signal breakdown with bar chart, wallet address, market name, trade volume, first deposit time, first trade time, and a Polygonscan link.

Live detections go to `#live-alerts`; backfill detections go to `#backfill-alerts` to avoid flooding the live channel with historical findings.

---

### 8. Redis Cache Layer

Redis is the shared cache across all consumer instances. Four key types:

| Key                    | Value                         | TTL       | Purpose                                                         |
| ---------------------- | ----------------------------- | --------- | --------------------------------------------------------------- |
| `deposit:{address}`    | First deposit timestamp       | 24 hours  | Subgraph result — never changes; prevents repeated GQL queries  |
| `token:{tokenID}`      | Market condition ID           | None      | Token→market mapping is immutable once a market is created      |
| `market:{conditionID}` | Market metadata struct        | 1 hour    | Gamma API result — one fetch serves all trades in that market   |
| `wallet:{address}`     | Wallet profile (trades/volume)| 5 minutes | Postgres query result — short TTL keeps it fresh during backfill|

Redis over in-memory: shared across consumer replicas, survives restarts with a warm cache, prevents thundering-herd on cold start.

---

### 9. Historical Backfill Service

A **run-once producer** (`cmd/backfill`) reads the seed wallet list (`backfill_wallets.yaml`), fetches all historical `OrderFilled` events via `eth_getLogs` in 100k-block chunks (semaphore-limited to 3 wallets in parallel), and publishes to `order-filled-backfill`. A separate **backfill consumer** — the same binary as the live consumer, configured via env vars — reads that topic and runs identical indexing and detection logic.

**Key design choices:**
- Same consumer binary for live and backfill avoids any scoring drift between paths.
- `backfill_progress` table makes the producer resumable — a crash restarts from the last completed chunk, not from block zero.
- Batch GraphQL query to The Graph at producer startup pre-warms Redis for all seed wallet deposits before any events are published — Step 1 is always a cache hit during backfill.
- Backfill alerts route to a separate Telegram channel; historical findings don't flood the live ops channel.

**False-positive control:** `0xc51eedc...` (Spotify scraper) is included in the seed list and must NOT be flagged. Its presence validates that the scoring model doesn't over-fire on high-volume non-insider wallets.

---

### 10. Telemetry & Analytics (Prometheus + Grafana)

Every stage of the event pipeline emits Prometheus metrics. All metric definitions live in `internal/metrics/metrics.go`; Kafka and Redis wrappers are self-instrumenting so callers get coverage automatically.

**Four Grafana dashboards:**

| Dashboard           | Key questions answered                                      |
| ------------------- | ----------------------------------------------------------- |
| Live Ingestion      | Is the listener keeping up? Is Kafka healthy?               |
| Detection Health    | Score distribution, alert rate, retry spike, cache hit rate |
| Backfill Progress   | Wallets complete, current block, ETA, consumer lag          |
| Infrastructure      | RPC/API latency, Redis pressure, subgraph sync lag          |

**Grafana alert rules:**

| Alert                    | Condition                                  | Severity | Impact                                      |
| ------------------------ | ------------------------------------------ | -------- | ------------------------------------------- |
| SubgraphSyncLagWarning   | Subgraph > 50 blocks behind chain tip      | warning  | Cache misses may return incomplete data     |
| SubgraphSyncLagCritical  | Subgraph > 200 blocks behind chain tip     | critical | High false-positive risk on wallet-age score|
| SubgraphIndexingError    | Subgraph reports hasIndexingErrors = true  | critical | Indexed data may be corrupt                 |
| KafkaConsumerLagHigh     | live-group lag > 1,000 messages            | warning  | Consumer falling behind live stream         |
| DetectionRetrySpike      | Retry publish rate > 1/s for 5 min        | warning  | Indexer likely slow; race condition elevated |

---

### 11. DevOps — Deployment, Monitoring & Escalation

#### Deployment

Three environments: local (Docker Compose), staging (VM with separate Telegram channel), production (VM). All services run as Docker containers with per-service restart policies — `listener` and `consumer` are `always`; `backfill-producer` is `on-failure` (exits 0 when done). Each container has a Docker `HEALTHCHECK` polling its `/metrics` endpoint.

Graceful shutdown on `SIGTERM`: stop consuming Kafka messages → drain detection channel → close DB pool → flush metrics → exit 0 (30-second grace period before SIGKILL).

#### CI/CD

GitHub Actions pipeline on every push to `main` and on PRs:

1. **Lint** — golangci-lint with errcheck, staticcheck, gosec
2. **Unit tests** — no external dependencies
3. **Integration tests** — testcontainers spins up real Postgres, Redis, Kafka
4. **Build** — Docker images tagged with git SHA
5. **Deploy** (main only) — SSH to VM, `docker compose pull`, `docker compose up -d`, post-deploy health check
6. **Rollback** — if health check fails, redeploy the previous image digest automatically and notify `#ops-alerts`

#### Log Management

All services emit structured JSON logs via `slog`. Log levels: DEBUG (disabled in prod), INFO (lifecycle events), WARN (retries, lag warnings), ERROR (write failures, send failures). Every log line carries `service`, `env`, `block_number`, and `wallet` fields where applicable.

Logs are shipped to **Loki** via Docker log driver and are queryable in Grafana Explore alongside metrics — no SSH needed for incident investigation.

#### Alerting & Escalation

```
Grafana PromQL alerts + Application Telegram alerts
        │
        v
   Alertmanager
        │
        ├── severity=warning  →  Telegram #ops-alerts  (no ack required)
        │
        └── severity=critical →  Telegram #ops-alerts  (immediate)
                                 + PagerDuty / OpsGenie after 5 min without ack
                                       │
                                       ├── Primary on-call  (immediate page)
                                       ├── Secondary on-call  (if no ack in 15 min)
                                       └── Team lead  (if no ack in 30 min)
```

Inhibition rules prevent double-paging: `SubgraphSyncLagCritical` suppresses `SubgraphSyncLagWarning`; `KafkaConsumerLagHigh` suppresses `DetectionRetrySpike` (lag causes retries — alert on the cause, not the symptom).

#### Alert Runbooks (summary)

| Alert                    | First action                                                                    |
| ------------------------ | ------------------------------------------------------------------------------- |
| SubgraphSyncLagCritical  | Check The Graph status; if sustained, set `DETECTION_SKIP_WALLET_AGE=true` on consumer to suppress false positives until subgraph recovers |
| KafkaConsumerLagHigh     | Check `detection_duration_seconds` p95 — if high, a subgraph/API call is blocking; consider scaling consumer replicas |
| DetectionRetrySpike      | Check `indexer_write_duration_seconds` — if high, Postgres is slow; increase `DETECTION_DELAY_MS` to give indexer time |
| Consumer crash           | `docker compose logs consumer` — Docker `always` policy auto-restarts; verify `last_confirmed_block` resumes within 60s |
| Backfill stalled         | Check `rpc_errors_total{service="backfill"}`; reduce `BACKFILL_CONCURRENCY` to 1 if RPC is rate-limiting |

#### Key Operational Config

All parameters are environment variables — no recompile needed to tune runtime behaviour.

| Variable                    | Default | Effect                                                             |
| --------------------------- | ------- | ------------------------------------------------------------------ |
| `CONFIRMATION_DEPTH`        | 10      | Polygon block depth before processing                              |
| `DETECTION_WORKERS`         | 4       | Detection goroutine pool size                                      |
| `DETECTION_DELAY_MS`        | 0       | Artificial delay before detection (gives indexer time to write)    |
| `DETECTION_SKIP_WALLET_AGE` | false   | Disable Signal 4 when subgraph is lagging — prevents false positives |
| `RETRY_MAX_ATTEMPTS`        | 5       | Max retries before dropping a stale detection                      |
| `BACKFILL_CONCURRENCY`      | 3       | Parallel wallet goroutines (reduce if RPC rate-limited)            |
| `BACKFILL_CHUNK_SIZE`       | 100000  | Blocks per getLogs chunk                                           |
| `SUBGRAPH_POLL_INTERVAL_S`  | 30      | Frequency of subgraph sync-lag health check                        |

---

## Tech Stack

| Component           | Technology                                                              |
| ------------------- | ----------------------------------------------------------------------- |
| Event Listener      | Go + go-ethereum (WebSocket)                                            |
| USDC.e Deposits     | The Graph subgraph (GraphQL) — on-demand, replaces eth_getLogs          |
| Message Broker      | Apache Kafka (3 topics)                                                 |
| Consumer Service    | Go (goroutines + channels) — single binary for live and backfill        |
| Backfill Producer   | Go batch job — getLogs for OrderFilled only, subgraph for deposits      |
| Database            | PostgreSQL 16                                                           |
| Cache               | Redis 7 (4 key types: deposit, token, market, wallet)                  |
| RPC Provider        | Alchemy / QuickNode (Polygon archive — OrderFilled only)                |
| Market Data         | Polymarket Gamma API + CLOB API                                         |
| Alerting            | Telegram Bot API + Alertmanager + PagerDuty / OpsGenie                 |
| Config              | Viper (env vars + config.yaml) + backfill_wallets.yaml                  |
| Logging             | slog (structured JSON) → Loki → Grafana                                 |
| Metrics             | Prometheus + Grafana (4 dashboards, 5 alert rules)                     |
| CI/CD               | GitHub Actions (lint → test → build → deploy → rollback)               |
| Containerisation    | Docker + Docker Compose                                                 |

---

## Directory Structure (abbreviated)

```
polymarket-insider-tracking/
├── cmd/
│   ├── listener/          # Event listener binary
│   ├── consumer/          # Live + backfill consumer binary (same binary, env-configured)
│   └── backfill/          # Backfill producer binary (run-once)
├── internal/
│   ├── listener/          # WebSocket + decoder + Kafka publisher
│   ├── consumer/          # Kafka consumer, indexer worker, retry consumer
│   ├── backfill/          # getLogs producer, progress tracking
│   ├── detection/         # Detector, 5 signal scorers, scorer, subgraph lookup, market lookup
│   ├── subgraph/          # The Graph GraphQL client (FirstDeposit, FirstDepositBatch, FetchMeta)
│   ├── cache/             # Redis read-through wrapper (self-instrumenting)
│   ├── alert/             # Telegram bot sender (self-instrumenting)
│   ├── store/             # Postgres CRUD (trades, wallets, markets, alerts, backfill)
│   ├── kafka/             # Producer + consumer wrappers (self-instrumenting)
│   ├── metrics/           # All Prometheus metric definitions
│   └── config/            # Viper configuration
├── scripts/
│   ├── tune_params.go     # Grid search → optimal weights → config.yaml
│   ├── validate.go        # Confusion matrix + per-wallet score report
│   └── testdata/          # positive_wallets.yaml + normal_wallets.yaml
├── migrations/            # SQL migration files
├── grafana/
│   ├── dashboards/        # 4 dashboard JSON files
│   └── alerts/alerts.yaml # 5 Grafana alert rule definitions
├── config.yaml            # Scoring weights + thresholds (written by tune_params.go)
├── backfill_wallets.yaml  # Seed insider wallets + false-positive control wallet
├── prometheus.yml         # Scrape config (4 targets)
├── docker-compose.yml
└── Makefile
```

---

## Data Flow Summary

**Live path:**
Polygon → WebSocket → Event Listener → Kafka `[order-filled]` → Consumer → Indexer Worker (Postgres) + Detection Channel → Redis/Subgraph lookups → score maker + taker → Telegram alert or DB update

**Backfill path:**
`backfill_wallets.yaml` → Backfill Producer → pre-warm Redis via The Graph batch query → `eth_getLogs` chunks → Kafka `[order-filled-backfill]` → Backfill Consumer (identical logic) → Telegram `#backfill-alerts` or DB update

**Race condition handling:**
If detection runs before the indexer writes the trade, the event is published to `[detection-retry]`. The retry consumer waits briefly and re-runs detection once the trade is confirmed in Postgres. Cache is already warm so Step 1 is always a cache hit on retry.
