# Polymarket Insider Detection — Mermaid Diagrams

Eight detailed flow diagrams covering every major subsystem. Each diagram is self-contained and can be read independently. The diagrams progress from the high-level architecture down to individual step-by-step flows.

---

## Diagram 1 — End-to-End System Architecture

Shows every component in the system and how they connect at a high level. Use this as the entry-point map when orienting to the codebase.

```mermaid
flowchart LR
    subgraph CHAIN["Polygon Network"]
        CTF["CTF Exchange<br/>0x4bFb41d5"]
        NRCTF["Neg Risk CTF<br/>0xC5d563A3"]
    end

    subgraph LISTENER["Event Listener  :9100/metrics"]
        WS["WebSocket<br/>10-block depth"]
        DEC["Decoder<br/>OrderFilled JSON"]
        ISTATE[("indexer_state<br/>restart point")]
    end

    subgraph KAFKA["Kafka"]
        KOF["order-filled<br/>4 partitions"]
        KOFB["order-filled-backfill<br/>2 partitions"]
        KDR["detection-retry<br/>2 partitions"]
    end

    subgraph CONSUMER["Consumer Service  :9101/metrics"]
        IW["Indexer Worker<br/>trades · wallets · markets"]
        DCH["Detection Channel<br/>scores maker + taker"]
        RC["Retry Consumer<br/>re-scores on race miss"]
    end

    subgraph BACKFILL["Backfill  :9102 prod / :9103 consumer"]
        BP["Backfill Producer<br/>run-once job"]
        BC["Backfill Consumer<br/>same binary, env-config"]
    end

    subgraph STORAGE["Storage"]
        PG[("PostgreSQL<br/>trades · wallets<br/>markets · alerts<br/>backfill_progress")]
        RD[("Redis<br/>deposit 24h<br/>token immutable<br/>market 1h<br/>wallet 5min")]
    end

    subgraph EXT["External Services"]
        SUBG["The Graph<br/>USDE.e Subgraph"]
        GAMMA["Polymarket<br/>Gamma API"]
    end

    subgraph ALERTS["Telegram Channels"]
        TLIVE["Live Alerts<br/>score >= 0.60"]
        TBKF["Backfill Alerts<br/>historical finds"]
        TOPS["#ops-alerts<br/>infra incidents"]
    end

    subgraph MON["Monitoring Stack"]
        PROM["Prometheus :9090<br/>15s scrape"]
        GRAF["Grafana :3000<br/>4 dashboards"]
        AM["Alertmanager<br/>5 alert rules"]
        PD["PagerDuty<br/>on-call rotation"]
    end

    CTF -->|"OrderFilled"| WS
    NRCTF -->|"OrderFilled"| WS
    WS --> DEC
    DEC --> ISTATE
    DEC -->|"JSON"| KOF

    KOF -->|"live-group"| IW
    KOF -->|"live-group"| DCH
    KDR --> RC
    RC --> DCH

    IW -->|"UPSERT"| PG
    DCH -->|"Step 3 miss"| PG
    DCH <-->|"Steps 1-3 cache"| RD
    DCH -->|"race → retry"| KDR
    DCH -->|"HIGH/MEDIUM"| TLIVE

    RD -->|"deposit miss"| SUBG
    SUBG -->|"SET deposit:addr"| RD
    RD -->|"market miss"| GAMMA
    GAMMA -->|"SET market:id"| RD
    RD -->|"market miss"| PG

    BP -->|"batch GraphQL pre-warm"| SUBG
    SUBG -->|"SET all deposit keys"| RD
    BP -->|"eth_getLogs"| KOFB
    KOFB -->|"backfill-group"| BC
    BC -->|"UPSERT"| PG
    BC -->|"historical find"| TBKF

    LISTENER -->|":9100/metrics"| PROM
    CONSUMER -->|":9101/metrics"| PROM
    BACKFILL -->|":9102/:9103/metrics"| PROM
    PROM -->|"PromQL"| GRAF
    GRAF --> AM
    AM -->|"warning"| TOPS
    AM -->|"critical + no ack 5min"| PD

    classDef chain fill:#fff3cd,stroke:#856404
    classDef kafka fill:#d1ecf1,stroke:#0c5460
    classDef storage fill:#d4edda,stroke:#155724
    classDef alert fill:#f8d7da,stroke:#721c24
    classDef mon fill:#e2d9f3,stroke:#5a189a
    classDef ext fill:#fce4ec,stroke:#880e4f

    class CTF,NRCTF chain
    class KOF,KOFB,KDR kafka
    class PG,RD storage
    class TLIVE,TBKF,TOPS,PD alert
    class PROM,GRAF,AM mon
    class SUBG,GAMMA ext
```

---

## Diagram 2 — Live Event Path: Block to Detection

Traces a single `OrderFilled` event from Polygon block arrival through Kafka to the Consumer, showing the exact handoff between the Indexer Worker and the Detection Channel.

```mermaid
flowchart TD
    A(["New block header<br/>arrives on WebSocket"]) --> B["Listener checks:<br/>block_number == latest - 10?"]
    B -->|"No — too recent"| A
    B -->|"Yes — confirmed block"| C["eth_getLogs for confirmed block<br/>CTF Exchange + Neg Risk CTF Exchange"]

    C --> D{"Events found?"}
    D -->|"None"| E["Update indexer_state.last_confirmed_block<br/>increment blocks_received_total"]
    D -->|"1+ events"| F["Decode each log<br/>ABI decode → OrderFilled struct"]

    F --> G["Build Kafka message JSON<br/>order_hash · maker · taker<br/>maker_asset_id · taker_asset_id<br/>maker_amount · taker_amount<br/>block_number · block_timestamp<br/>tx_hash · log_index · exchange"]

    G --> H["Publish to Kafka<br/>topic: order-filled<br/>partitioned by tx_hash"]

    H --> I{"Kafka publish<br/>succeeded?"}
    I -->|"Error"| J["Retry with exponential backoff<br/>increment kafka_publish_errors_total"]
    J --> H
    I -->|"OK"| K["Update indexer_state<br/>increment kafka_messages_published_total"]

    K --> L["Consumer reads from<br/>order-filled topic<br/>live-group, autocommit off"]
    L --> M["Fan-out: send event to<br/>both goroutines concurrently"]

    M --> N["Indexer Worker<br/>UPSERT into trades table<br/>UNIQUE(tx_hash, log_index)<br/>ON CONFLICT DO NOTHING"]
    M --> O["Detection Channel<br/>buffered Go channel<br/>(depth monitored as gauge)"]

    N -->|"Commit Kafka offset<br/>after both goroutines ack"| P(["Event fully processed"])
    O --> Q["Detection goroutine<br/>scores maker, then taker<br/>See Diagram 3"]

    classDef polygon fill:#fff3cd,stroke:#856404
    classDef kafka fill:#d1ecf1,stroke:#0c5460
    classDef consumer fill:#d4edda,stroke:#155724
    classDef decision fill:#e2d9f3,stroke:#5a189a

    class A,B,C polygon
    class G,H,I,J,K kafka
    class L,M,N,O consumer
```

---

## Diagram 3 — Detection Algorithm: Dual Maker/Taker Scoring

Every `OrderFilled` event is scored twice — once for `maker`, once for `taker`. Each run is fully independent. Steps 1–5 run identically for both addresses; only the market metadata cache (Step 2) is shared between the two runs.

```mermaid
flowchart TD
    START(["OrderFilled event<br/>received on Detection Channel"]) --> SPLIT["Extract both parties:<br/>maker address<br/>taker address"]

    SPLIT --> MAKER["Run Steps 1-5<br/>for MAKER address"]
    SPLIT --> TAKER["Run Steps 1-5<br/>for TAKER address"]

    subgraph MAKER_FLOW["Maker Scoring  (passive limit-order placer)"]
        M1["Step 1: USDC.e First Deposit<br/>Redis deposit:maker → The Graph"]
        M2["Step 2: Market Metadata<br/>Redis token:id → market:conditionID<br/>(shared cache with taker run)"]
        M3["Step 3: Wallet Profile<br/>Redis wallet:maker → Postgres"]
        M4["Step 4: Score 5 signals<br/>Entry Timing · Market Count<br/>Trade Size · Wallet Age · Concentration"]
        M5["Step 5: Aggregate Score<br/>0.25*timing + 0.20*count<br/>+ 0.20*size + 0.15*age + 0.20*conc"]
        MROUTE{"Maker<br/>aggregate score"}
    end

    subgraph TAKER_FLOW["Taker Scoring  (active order filler — more likely insider)"]
        T1["Step 1: USDC.e First Deposit<br/>Redis deposit:taker → The Graph"]
        T2["Step 2: Market Metadata<br/>Redis token:id → market:conditionID<br/>(same cache entry as maker run)"]
        T3["Step 3: Wallet Profile<br/>Redis wallet:taker → Postgres"]
        T4["Step 4: Score 5 signals<br/>Entry Timing · Market Count<br/>Trade Size · Wallet Age · Concentration"]
        T5["Step 5: Aggregate Score<br/>0.25*timing + 0.20*count<br/>+ 0.20*size + 0.15*age + 0.20*conc"]
        TROUTE{"Taker<br/>aggregate score"}
    end

    MAKER --> M1 --> M2 --> M3 --> M4 --> M5 --> MROUTE
    TAKER --> T1 --> T2 --> T3 --> T4 --> T5 --> TROUTE

    MROUTE -->|"score >= 0.80"| MHIGH["HIGH alert<br/>Telegram send<br/>Write to alerts table"]
    MROUTE -->|"0.60 <= score < 0.80"| MMED["MEDIUM flag<br/>Telegram send<br/>Write to alerts table"]
    MROUTE -->|"score < 0.60"| MLOW["LOW<br/>Update wallets table<br/>not_insider = true"]

    TROUTE -->|"score >= 0.80"| THIGH["HIGH alert<br/>Telegram send<br/>Write to alerts table"]
    TROUTE -->|"0.60 <= score < 0.80"| TMED["MEDIUM flag<br/>Telegram send<br/>Write to alerts table"]
    TROUTE -->|"score < 0.60"| TLOW["LOW<br/>Update wallets table<br/>not_insider = true"]

    classDef step fill:#d1ecf1,stroke:#0c5460
    classDef route fill:#e2d9f3,stroke:#5a189a
    classDef high fill:#f8d7da,stroke:#721c24
    classDef low fill:#d4edda,stroke:#155724

    class M1,M2,M3,M4,M5,T1,T2,T3,T4,T5 step
    class MHIGH,THIGH high
    class MLOW,TLOW low
```

---

## Diagram 4 — Detection Steps 1–5: Signal Detail

Step-by-step flow within a single address scoring run (same logic for both maker and taker). Shows every cache hit/miss path, the 5 signal formulas, and the final threshold routing with retry logic.

```mermaid
flowchart TD
    IN(["Start: score address<br/>for OrderFilled event"]) --> S1

    subgraph STEP1["Step 1 — USDC.e First Deposit  (wallet age signal input)"]
        S1["Redis GET deposit:address"] --> S1HIT{"Cache hit?"}
        S1HIT -->|"HIT ~1ms"| S1USE["Use cached first-deposit timestamp<br/>increment cache_hits_total{deposit}"]
        S1HIT -->|"MISS ~50-150ms"| S1SUBG["Query The Graph subgraph<br/>transfers(where:{to:addr}, first:1)"]
        S1SUBG --> S1SET["SET Redis deposit:address<br/>TTL 24h<br/>increment cache_misses_total{deposit}"]
        S1SET --> S1USE
    end

    subgraph STEP2["Step 2 — Market Metadata"]
        S2A["Redis GET token:tokenID"] --> S2AHIT{"Token cache hit?"}
        S2AHIT -->|"HIT"| S2B["Redis GET market:conditionID"]
        S2AHIT -->|"MISS"| S2PG["Query Postgres markets table<br/>SET Redis token:tokenID (immutable)"]
        S2PG --> S2B
        S2B --> S2BHIT{"Market cache hit?"}
        S2BHIT -->|"HIT ~1ms"| S2USE["Use cached market<br/>end_date · category · total_volume"]
        S2BHIT -->|"MISS"| S2GAMMA["Fetch from Gamma API<br/>Store in Postgres<br/>SET Redis market:id TTL 1h"]
        S2GAMMA --> S2USE
    end

    subgraph STEP3["Step 3 — Wallet Profile"]
        S3["Redis GET wallet:address"] --> S3HIT{"Cache hit?"}
        S3HIT -->|"HIT ~1ms"| S3USE["Use cached profile<br/>total_trades · distinct_markets<br/>total_volume · concentration"]
        S3HIT -->|"MISS"| S3PG["Query Postgres:<br/>trade count · market count<br/>total volume · max-market ratio<br/>SET Redis wallet:addr TTL 5min"]
        S3PG --> S3USE
    end

    subgraph STEP4["Step 4 — Score 5 Signals"]
        SIG1["Signal 1: Entry Timing  w=0.25<br/>time_to_resolution = end_date - trade_ts<br/>score = exp(-hours/12)<br/>max score if <= 1h before resolution"]
        SIG2["Signal 2: Market Count  w=0.20<br/>1 market → 1.0<br/>2 markets → 0.8<br/>3 markets → 0.5<br/>>5 markets → 0.0"]
        SIG3["Signal 3: Trade Size  w=0.20<br/>abs = min(volume/10000, 1.0)<br/>rel = min(wallet_vol/market_vol/0.25, 1.0)<br/>score = max(abs, rel)"]
        SIG4["Signal 4: Wallet Age  w=0.15<br/>age = first_trade - first_deposit<br/>score = exp(-age_hours/6)<br/>max score if <= 10 min"]
        SIG5["Signal 5: Concentration  w=0.20<br/>conc = max_market_vol / total_wallet_vol<br/>>= 0.95 → 1.0<br/>>= 0.80 → 0.7<br/>>= 0.60 → 0.3"]
    end

    subgraph STEP5["Step 5 — Aggregate & Route"]
        AGG["Weighted aggregate:<br/>0.25*timing + 0.20*count<br/>+ 0.20*size + 0.15*age<br/>+ 0.20*concentration"]
        THR{"Score<br/>threshold"}
        HIGH["score >= 0.80<br/>HIGH — auto-flag<br/>Telegram alert sent"]
        MED["0.60 <= score < 0.80<br/>MEDIUM — flag for review<br/>Telegram alert sent"]
        LOW["score < 0.60<br/>LOW — not insider<br/>DB updated"]
        RCHECK{"Indexer race?<br/>wallet not in DB yet"}
        RETRY["Publish to<br/>detection-retry topic<br/>retry after indexer catches up"]
    end

    S1USE --> S2A
    S2USE --> S3
    S3USE --> SIG1
    S3USE --> SIG2
    S3USE --> SIG3
    S3USE --> SIG4
    S3USE --> SIG5
    SIG1 --> AGG
    SIG2 --> AGG
    SIG3 --> AGG
    SIG4 --> AGG
    SIG5 --> AGG
    AGG --> THR
    THR -->|">= 0.80"| HIGH
    THR -->|"0.60-0.79"| MED
    THR -->|"< 0.60"| RCHECK
    RCHECK -->|"Race detected"| RETRY
    RCHECK -->|"Normal"| LOW

    classDef cache fill:#d1ecf1,stroke:#0c5460
    classDef signal fill:#fff3cd,stroke:#856404
    classDef high fill:#f8d7da,stroke:#721c24
    classDef low fill:#d4edda,stroke:#155724

    class S1,S1SUBG,S1SET,S2A,S2PG,S2B,S2GAMMA,S3,S3PG cache
    class SIG1,SIG2,SIG3,SIG4,SIG5,AGG signal
    class HIGH,MED high
    class LOW low
```

---

## Diagram 5 — Redis Cache Read-Through Pattern

How the four Redis key types work. Each key type has a different TTL strategy and a different fallback chain when the cache misses.

```mermaid
flowchart TD
    REQ(["Detection goroutine<br/>needs data for address/token/market"]) --> WHICH{"Which<br/>data type?"}

    WHICH -->|"First USDC.e deposit"| DEP
    WHICH -->|"Token → Market mapping"| TOK
    WHICH -->|"Market metadata"| MKT
    WHICH -->|"Wallet profile"| WAL

    subgraph DEP_FLOW["deposit:address  — 24h TTL"]
        DEP["Redis GET deposit:addr"]
        DEP --> DEPHIT{"Hit?"}
        DEPHIT -->|"HIT<br/>~1ms"| DEPUSE["Return Unix timestamp<br/>first USDC.e deposit"]
        DEPHIT -->|"MISS<br/>~50-150ms"| DEPGRAPH["The Graph GraphQL:<br/>transfers(where:{to:addr}, first:1)"]
        DEPGRAPH --> DEPSET["SET Redis deposit:addr<br/>TTL 24h<br/>First deposit never changes"]
        DEPSET --> DEPUSE
    end

    subgraph TOK_FLOW["token:tokenID  — No TTL (immutable)"]
        TOK["Redis GET token:tokenID"]
        TOK --> TOKHIT{"Hit?"}
        TOKHIT -->|"HIT<br/>~1ms"| TOKUSE["Return conditionID<br/>(no expiry needed —<br/>tokenID→market is permanent)"]
        TOKHIT -->|"MISS"| TOKPG["Postgres markets table<br/>lookup by token_id"]
        TOKPG --> TOKGAMMA{"Found<br/>in DB?"}
        TOKGAMMA -->|"Yes"| TOKSET["SET Redis token:tokenID<br/>no TTL"]
        TOKGAMMA -->|"No"| TOKAPI["Fetch from Gamma API<br/>Store in Postgres<br/>SET Redis token:tokenID"]
        TOKSET --> TOKUSE
        TOKAPI --> TOKUSE
    end

    subgraph MKT_FLOW["market:conditionID  — 1h TTL"]
        MKT["Redis GET market:conditionID"]
        MKT --> MKTHIT{"Hit?"}
        MKTHIT -->|"HIT<br/>~1ms"| MKTUSE["Return Market struct<br/>end_date · category<br/>total_volume · resolution"]
        MKTHIT -->|"MISS"| MKTPG["Postgres markets table<br/>lookup by condition_id"]
        MKTPG --> MKTFOUND{"Found<br/>in DB?"}
        MKTFOUND -->|"Yes"| MKTSET["SET Redis market:conditionID<br/>TTL 1h<br/>(volume changes — refresh hourly)"]
        MKTFOUND -->|"No"| MKTAPI["Fetch from Gamma API<br/>Upsert into Postgres<br/>SET Redis market:conditionID<br/>TTL 1h"]
        MKTSET --> MKTUSE
        MKTAPI --> MKTUSE
    end

    subgraph WAL_FLOW["wallet:address  — 5min TTL"]
        WAL["Redis GET wallet:addr"]
        WAL --> WALHIT{"Hit?"}
        WALHIT -->|"HIT<br/>~1ms"| WALUSE["Return wallet profile:<br/>total_trades · distinct_markets<br/>total_volume · concentration"]
        WALHIT -->|"MISS"| WALPG["Postgres:<br/>SELECT COUNT trades, COUNT markets,<br/>SUM volume, MAX market vol<br/>for this wallet address"]
        WALPG --> WALSET["SET Redis wallet:addr<br/>TTL 5min<br/>(short: indexer writes trades<br/>concurrently — stay fresh)"]
        WALSET --> WALUSE
    end

    classDef cache fill:#d1ecf1,stroke:#0c5460
    classDef ext fill:#fce4ec,stroke:#880e4f
    classDef db fill:#d4edda,stroke:#155724

    class DEP,TOK,MKT,WAL,DEPSET,TOKSET,MKTSET,WALSET cache
    class DEPGRAPH,TOKAPI,MKTAPI ext
    class TOKPG,MKTPG,WALPG db
```

---

## Diagram 6 — Historical Backfill Pipeline

The backfill is a run-once producer job plus a long-running consumer (same binary as live, different env config). It pre-warms Redis at startup, then feeds historical `OrderFilled` events through the same detection logic as the live path.

```mermaid
flowchart TD
    YAML(["backfill_wallets.yaml<br/>7 known insiders<br/>+ false-positive controls"]) --> BP_START

    subgraph PRODUCER["Backfill Producer  cmd/backfill/main.go  :9102/metrics"]
        BP_START["Load seed wallet list<br/>Resume state from backfill_progress table<br/>(idempotent restart)"]

        BP_START --> PREWARM["Batch GraphQL to The Graph<br/>FirstDepositBatch(wallets: all_seed_addrs)<br/>Single round-trip ~150ms"]
        PREWARM --> REDIS_SEED["SET Redis deposit:addr  TTL 24h<br/>for all seed wallets<br/>(pre-warms cache before getLogs starts)"]

        REDIS_SEED --> SEM["Semaphore: N=3 concurrent wallets<br/>(avoids RPC rate-limiting)"]
        SEM --> WALLET_LOOP["For each wallet (goroutine)"]

        WALLET_LOOP --> BLOCK_RANGE["Block range strategy:<br/>Start: contract deploy block ~32,600,000<br/>End: latest - 10<br/>Chunk size: 100,000 blocks ~55h"]

        BLOCK_RANGE --> GETLOGS["eth_getLogs per chunk:<br/>CTF Exchange + Neg Risk CTF Exchange<br/>topics[2]=maker_addr UNION topics[3]=taker_addr"]

        GETLOGS --> DECODE["Decode each log → OrderFilled struct<br/>(same ABI decoder as live listener)"]
        DECODE --> PUBLISH["Publish to Kafka:<br/>topic: order-filled-backfill<br/>same JSON schema as order-filled"]
        PUBLISH --> PROGRESS["UPDATE backfill_progress<br/>last_block = chunk_end<br/>status = in_progress"]
        PROGRESS --> MORE{"More<br/>chunks?"}
        MORE -->|"Yes"| GETLOGS
        MORE -->|"No"| DONE["UPDATE backfill_progress<br/>status = complete"]
        DONE --> NEXT{"More<br/>wallets?"}
        NEXT -->|"Yes"| WALLET_LOOP
        NEXT -->|"No"| EXIT["Producer exits 0<br/>Docker on-failure restart<br/>will not restart after clean exit"]
    end

    subgraph CONSUMER_BC["Backfill Consumer  (same binary)  :9103/metrics"]
        BC_CONF["Env config:<br/>KAFKA_TOPIC=order-filled-backfill<br/>KAFKA_GROUP_ID=backfill-group<br/>TELEGRAM_CHAT_ID=BACKFILL_CHAT_ID"]
        BC_CONF --> BC_CONSUME["Consume from order-filled-backfill<br/>(backfill-group)"]
        BC_CONSUME --> BC_INDEX["Indexer Worker<br/>UPSERT UNIQUE(tx_hash, log_index)<br/>ON CONFLICT DO NOTHING<br/>(safe overlap with live path)"]
        BC_CONSUME --> BC_DETECT["Detection Channel<br/>Steps 1-5 identical to live path<br/>Same scorer.go, same signals"]
        BC_DETECT -->|"score >= 0.60"| TG_HIST["Telegram Backfill Channel<br/>Historical insider finding<br/>(separate from live alerts)"]
        BC_DETECT -->|"score < 0.60"| BC_DB["Update wallets table<br/>not_insider = true"]
        BC_INDEX --> BC_DB
    end

    PUBLISH --> BC_CONSUME

    classDef prod fill:#d1ecf1,stroke:#0c5460
    classDef consumer fill:#d4edda,stroke:#155724
    classDef alert fill:#f8d7da,stroke:#721c24

    class BP_START,PREWARM,REDIS_SEED,SEM,WALLET_LOOP,BLOCK_RANGE,GETLOGS,DECODE,PUBLISH,PROGRESS prod
    class BC_CONF,BC_CONSUME,BC_INDEX,BC_DETECT consumer
    class TG_HIST alert
```

---

## Diagram 7 — Telemetry: Metrics Collection Across All Services

Every stage of the event lifecycle emits structured Prometheus metrics. This diagram shows the 10 instrumented stages (L1–L10) from block arrival through detection to alerting, and the separate backfill path stages (B1–B3).

```mermaid
flowchart TD
    subgraph LISTENER["Event Listener  :9100/metrics"]
        L1["L1 Block Header Received<br/>blocks_received_total<br/>last_confirmed_block gauge"]
        L2["L2 getLogs + Decode<br/>rpc_calls_total<br/>rpc_call_duration_seconds<br/>events_decoded_total"]
        L3["L3 Kafka Publish → order-filled<br/>kafka_messages_published_total<br/>kafka_publish_duration_seconds"]
    end

    subgraph CONSUMER_LIVE["Consumer Service  :9101/metrics"]
        L4["L4 Kafka Consume<br/>kafka_messages_consumed_total<br/>kafka_consumer_lag gauge<br/>kafka_consume_lag_seconds"]
        L5A["L5a Indexer Worker<br/>trades_indexed_total<br/>indexer_write_duration_seconds"]
        L5B["L5b Detection Channel enqueue<br/>detection_channel_depth gauge"]
        L6["L6 Redis Cache Lookups<br/>cache_hits_total {token,market,deposit,wallet}<br/>cache_misses_total<br/>cache_lookup_duration_seconds"]
        L7["L7 External Calls on cache miss<br/>subgraph_queries_total<br/>subgraph_query_duration_seconds<br/>api_calls_total {gamma}<br/>api_call_duration_seconds"]
        L8["L8 Score 5 Signals<br/>signal_score histogram {per signal}<br/>aggregate_score histogram"]
        L9["L9 Detection Routing<br/>detection_runs_total {flagged_high/medium/not_insider/retry}<br/>detection_duration_seconds<br/>event_e2e_duration_seconds"]
        L10A["L10a Telegram Alert Send<br/>telegram_sends_total {channel, severity}<br/>telegram_send_duration_seconds"]
        L10B["L10b Kafka Retry Publish<br/>detection_retries_published_total<br/>kafka_publish_duration_seconds"]
        BG1["Background Goroutine every 30s<br/>subgraph_last_indexed_block gauge<br/>subgraph_indexing_errors_total<br/>via The Graph _meta query"]
    end

    subgraph BACKFILL_SVC["Backfill Services  :9102/:9103/metrics"]
        B1["B1 Backfill getLogs per chunk<br/>rpc_calls_total {backfill}<br/>backfill_chunk_duration_seconds<br/>backfill_current_block gauge"]
        B2["B2 Backfill Kafka Publish<br/>kafka_messages_published_total {order-filled-backfill}<br/>kafka_publish_duration_seconds"]
        B3["B3 Backfill Progress<br/>backfill_wallets_total gauge<br/>backfill_wallets_complete gauge<br/>backfill_wallets_failed gauge<br/>backfill_wallet_duration_seconds"]
        B4["B4 Backfill Consumer Lag<br/>kafka_consumer_lag {backfill-group}<br/>kafka_consume_lag_seconds"]
    end

    subgraph MON_STACK["Monitoring Stack"]
        PROM["Prometheus :9090<br/>scrapes all 4 services every 15s<br/>retains 15-day TSDB"]
        GRAF["Grafana :3000<br/>4 dashboards<br/>Dash 1: Live Ingestion<br/>Dash 2: Detection Health<br/>Dash 3: Backfill Progress<br/>Dash 4: Infrastructure"]
        AM["Alertmanager<br/>5 PromQL alert rules"]
        RULES["Alert Rules:<br/>SubgraphSyncLagWarning >50 blocks<br/>SubgraphSyncLagCritical >200 blocks<br/>SubgraphIndexingError rate>0<br/>KafkaConsumerLagHigh >1000 msgs<br/>DetectionRetrySpike rate>1/s"]
    end

    L1 --> L2 --> L3 --> L4
    L4 --> L5A
    L4 --> L5B --> L6 --> L7 --> L8 --> L9
    L9 --> L10A
    L9 --> L10B
    B1 --> B2 --> B3
    B4 -.->|"backfill consumer lag"| PROM

    LISTENER -->|":9100/metrics"| PROM
    CONSUMER_LIVE -->|":9101/metrics"| PROM
    BACKFILL_SVC -->|":9102/:9103/metrics"| PROM

    PROM --> GRAF
    GRAF --> AM
    AM --> RULES

    classDef listener fill:#fff3cd,stroke:#856404
    classDef consumer fill:#d1ecf1,stroke:#0c5460
    classDef backfill fill:#d4edda,stroke:#155724
    classDef mon fill:#e2d9f3,stroke:#5a189a

    class L1,L2,L3 listener
    class L4,L5A,L5B,L6,L7,L8,L9,L10A,L10B,BG1 consumer
    class B1,B2,B3,B4 backfill
    class PROM,GRAF,AM,RULES mon
```

---

## Diagram 8 — Alert Escalation Chain

Two alert sources (Grafana infrastructure alerts and application Telegram alerts) are routed through a tiered escalation chain with inhibition rules and on-call rotation.

```mermaid
flowchart TD
    subgraph SOURCES["Alert Sources"]
        GRAFANA_ALERTS["Grafana Alert Rules (PromQL)<br/><br/>SubgraphSyncLagWarning<br/>last_confirmed_block - subgraph_last_indexed_block > 50<br/>for 1m  — severity: warning<br/><br/>SubgraphSyncLagCritical<br/>last_confirmed_block - subgraph_last_indexed_block > 200<br/>for 2m  — severity: critical<br/><br/>SubgraphIndexingError<br/>rate(subgraph_indexing_errors_total[5m]) > 0<br/>for 1m  — severity: critical<br/><br/>KafkaConsumerLagHigh<br/>kafka_consumer_lag{topic=order-filled,group=live-group} > 1000<br/>for 2m  — severity: warning<br/><br/>DetectionRetrySpike<br/>rate(detection_retries_published_total[5m]) > 1<br/>for 5m  — severity: warning"]

        APP_ALERTS["Application Alerts (direct Telegram Bot)<br/><br/>Insider Detected<br/>score >= 0.60 — Live Channel<br/><br/>Historical Insider Found<br/>score >= 0.60 — Backfill Channel<br/><br/>Telegram send failure<br/>error logged + retry"]
    end

    subgraph INHIBIT["Inhibition Rules (no double-paging)"]
        INH1["SubgraphSyncLagCritical<br/>inhibits<br/>SubgraphSyncLagWarning<br/>(don't page warning if critical already firing)"]
        INH2["KafkaConsumerLagHigh<br/>inhibits<br/>DetectionRetrySpike<br/>(lag causes retries — alert on root cause only)"]
    end

    subgraph AM["Alertmanager"]
        ROUTE{"Route by<br/>severity"}
    end

    subgraph WARN_PATH["Warning Path"]
        W1["Telegram #ops-alerts<br/>Fire immediately<br/>Repeat every 4h<br/>Group wait 30s<br/>No acknowledgement required"]
    end

    subgraph CRIT_PATH["Critical Path"]
        C1["Telegram #ops-alerts<br/>Fire immediately<br/>Repeat every 1h<br/>Group wait 10s"]
        C2{"Acknowledged<br/>within 5 min?"}
        C3["PagerDuty / ZenDesk<br/>Create incident<br/>Page Primary on-call"]
        C4{"Primary acks<br/>within 15 min?"}
        C5["Page Secondary on-call"]
        C6{"Secondary acks<br/>within 30 min?"}
        C7["Escalate to Team Lead<br/>Send summary to<br/>#ops-alerts channel"]
    end

    subgraph RUNBOOKS["Alert Runbooks (linked from Grafana annotation)"]
        RB1["SubgraphSyncLagCritical:<br/>1. Check The Graph status page<br/>2. If sustained >10min: set DETECTION_SKIP_WALLET_AGE=true<br/>   restart consumer (disables wallet-age signal)<br/>3. Once lag clears: remove override, restart"]
        RB2["KafkaConsumerLagHigh:<br/>1. Check detection_duration_seconds p95<br/>2. If >5s: subgraph/Gamma API blocking<br/>3. If CPU-bound: scale consumer replicas<br/>   (increase partitions first)"]
        RB3["DetectionRetrySpike:<br/>1. Check indexer_write_duration_seconds<br/>2. If high Postgres latency: set DETECTION_DELAY_MS=500<br/>3. Check indexer_errors_total for constraint violations"]
        RB4["Consumer Crash:<br/>1. docker compose logs consumer --tail=100<br/>2. Docker always restart will auto-recover<br/>3. Verify last_confirmed_block resumes within 60s"]
        RB5["Backfill Stalled:<br/>1. Check backfill_current_block per wallet<br/>2. Reduce BACKFILL_CONCURRENCY from 3 to 1<br/>3. Check rpc_errors_total{service=backfill}"]
    end

    GRAFANA_ALERTS --> INHIBIT
    INHIBIT --> AM
    APP_ALERTS -->|"direct Telegram Bot API<br/>bypasses Alertmanager"| W1

    ROUTE -->|"severity=warning"| W1
    ROUTE -->|"severity=critical"| C1

    C1 --> C2
    C2 -->|"Yes — resolved"| RESOLVED(["Incident closed"])
    C2 -->|"No — escalate"| C3
    C3 --> C4
    C4 -->|"Acks"| RESOLVED
    C4 -->|"No ack"| C5
    C5 --> C6
    C6 -->|"Acks"| RESOLVED
    C6 -->|"No ack"| C7

    AM --> ROUTE

    C1 -.->|"runbook link"| RB1
    W1 -.->|"runbook links"| RB2
    W1 -.->|"runbook links"| RB3
    C3 -.->|"runbook links"| RB4
    C3 -.->|"runbook links"| RB5

    classDef source fill:#d1ecf1,stroke:#0c5460
    classDef warn fill:#fff3cd,stroke:#856404
    classDef crit fill:#f8d7da,stroke:#721c24
    classDef runbook fill:#e2d9f3,stroke:#5a189a
    classDef ok fill:#d4edda,stroke:#155724

    class GRAFANA_ALERTS,APP_ALERTS source
    class W1,INH1,INH2 warn
    class C1,C2,C3,C4,C5,C6,C7 crit
    class RB1,RB2,RB3,RB4,RB5 runbook
    class RESOLVED ok
```

---

## Quick Reference: Diagram Index

| # | Diagram | Key Question Answered |
|---|---------|----------------------|
| 1 | End-to-End System Architecture | What are all the components and how do they connect? |
| 2 | Live Event Path: Block to Detection | How does one `OrderFilled` event flow from Polygon to the detection goroutine? |
| 3 | Detection Algorithm: Dual Maker/Taker Scoring | Why are both maker and taker scored, and what are the outcomes? |
| 4 | Detection Steps 1–5: Signal Detail | What happens inside a single address scoring run, step by step? |
| 5 | Redis Cache Read-Through Pattern | How does each of the 4 Redis key types work with its fallback chain? |
| 6 | Historical Backfill Pipeline | How does the backfill producer seed Redis and feed the same consumer? |
| 7 | Telemetry: Metrics Collection | What is instrumented at each stage and how does it reach Grafana? |
| 8 | Alert Escalation Chain | How does a PromQL threshold turn into a PagerDuty page? |
