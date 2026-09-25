<!-- anchor: README.md:L1-L100 sha:HEAD -->

# Market Data Gateway (MDG) Overview

The **Market Data Gateway (MDG)** (`@astrophage/market-data-gateway`) is a high-throughput, ultra-low-latency edge dissemination service within the **Global Financial Markets Group (GFMG) Nexus Trading Exchange (NTE)** platform. Serving as the primary egress boundary between internal trading operations and external market participants, MDG bridges backend distributed streaming infrastructure with client-facing WebSocket connections.

---

## 1. System Topology & Architecture

MDG implements an asynchronous, event-driven pipeline that decouples internal message brokers from external client connections. By isolating broker partitions from connection stalls, the gateway maintains market fairness and deterministic distribution latencies across all subscribed endpoints.

```
                   +------------------------------------+
                   |     [[entities/order-matching-engine|Order Matching Engine]]      |
                   +------------------------------------+
                                      |
              +-----------------------+-----------------------+
              | (nte.orderbook.snapshots)                     | (nte.trades.matched)
              v                                               v
+-----------------------------+               +-----------------------------+
|    [[entities/order-book-consumer|OrderBookConsumer]]    |               |      [[entities/trade-consumer|TradeConsumer]]      |
|  (grp: mdg-orderbook-group) |               |    (grp: mdg-trade-group)   |
+-----------------------------+               +-----------------------------+
              \                                               /
               \                                             /
                v                                           v
      +---------------------------------------------------------------+
      |                      [[entities/client-manager|ClientManager]]                        |
      |          (WebSocket Connection Pool & Fan-out Engine)         |
      +---------------------------------------------------------------+
                                      |
                    (JSON / L2_UPDATE & TRADE_TICK)
                                      v
                      +-------------------------------+
                      | External Clients / UI Egress  |
                      +-------------------------------+
```

For deeper architectural context on the end-to-end data path, see [[concepts/data-lifecycle]] and [[concepts/ingestion-egress-decoupling]].

---

## 2. Core Components

The MDG codebase is organized into four modular layers:

### 2.1. Bootstrap & Runtime Orchestration (`src/index.ts`)
* Acts as the application entry point.
* Instantiates the [[entities/client-manager|ClientManager]] WebSocket server on TCP port `8080`.
* Configures and starts the stream ingestion consumers: [[entities/order-book-consumer|OrderBookConsumer]] and [[entities/trade-consumer|TradeConsumer]].
* Manages runtime lifecycle events and orchestrates graceful shutdowns on process termination.

### 2.2. Ingestion Subsystem (`src/consumers/`)
The ingestion layer connects to [[entities/kafka-broker|Kafka]] brokers (configured via `KAFKA_BROKERS`, defaulting to `kafka1.nte.internal:9092` in production) to stream execution events:
* **[[entities/order-book-consumer|OrderBookConsumer]]**: Connects with client ID `mdg-ob-consumer` under consumer group `mdg-orderbook-group`. It ingests aggregated Level 2 depth snapshots from the `nte.orderbook.snapshots` topic.
* **[[entities/trade-consumer|TradeConsumer]]**: Connects with client ID `mdg-trade-consumer` under consumer group `mdg-trade-group`. It ingests trade execution ticks from the `nte.trades.matched` topic.

See [[decisions/independent-consumer-groups]] for the design rationale behind isolating these consumer groups.

### 2.3. Dissemination & Connection Subsystem (`src/websockets/ClientManager.ts`)
* Manages the lifecycle of active client connections using the `ws` library.
* Maintains an in-memory `Set<WebSocket>` connection pool.
* Performs non-blocking fan-out broadcasts to connected clients:
  * `broadcastOrderBook(snapshot)`: Wraps snapshots into `{ type: "L2_UPDATE", data: snapshot }`.
  * `broadcastTrade(trade)`: Wraps trade ticks into `{ type: "TRADE_TICK", data: trade }`.
* Filters out inactive sockets by verifying `client.readyState === WebSocket.OPEN` before writes to prevent memory leaks and thread blocking.

Detailed mechanics are covered in [[concepts/websocket-broadcasting]] and [[decisions/low-latency-egress-design]].

### 2.4. Contract & Data Modeling Layer (`protos/` & `src/models/`)
* **Protobuf Wire Contracts (`protos/market_data.proto`)**: Defines the protocol schemas published by the [[entities/order-matching-engine|Order Matching Engine]] and shared across NTE services:
  * `nte.marketdata.OrderBookSnapshot`: Encapsulates `symbol`, `timestamp_ns`, and repeated `Level` objects (`price`, `quantity`, `order_count`).
  * `nte.marketdata.TradeTick`: Encapsulates `trade_id`, `symbol`, `price`, `quantity`, `matched_at_ns`, and `taker_side`.
  * See [[entities/protobuf-contracts]] for the wire definitions.
* **TypeScript Runtime Types (`src/models/types.ts`)**: Defines strongly typed runtime data models including `OrderBookEvent`, `TradeEvent`, and `PriceLevel`. Detailed descriptions are in [[entities/runtime-models]].

---

## 3. End-to-End Execution Flow

```
[Matching Core Engine]
        │
        ├──► Kafka: `nte.orderbook.snapshots` ──► OrderBookConsumer ──► ClientManager.broadcastOrderBook ──► WS Clients (L2_UPDATE)
        │
        └──► Kafka: `nte.trades.matched`      ──► TradeConsumer     ──► ClientManager.broadcastTrade     ──► WS Clients (TRADE_TICK)
```

1. **Generation**: The [[entities/order-matching-engine|Order Matching Engine]] executes order matches and compiles resting limit book updates into L2 depth snapshots.
2. **Ingestion**: [[entities/order-book-consumer|OrderBookConsumer]] and [[entities/trade-consumer|TradeConsumer]] asynchronously poll Kafka partitions.
3. **Packaging**: Ingested payloads are structured into standardized event envelopes (`L2_UPDATE` or `TRADE_TICK`).
4. **State Caching & Hydration**: Market state is cached in [[entities/redis-cache|Redis]] for instant recovery and connection hydration (see [[concepts/state-hydration]] and [[decisions/redis-l2-state-caching]]).
5. **Fan-Out Egress**: [[entities/client-manager|ClientManager]] iterates over the active WebSocket pool and dispatches JSON payloads to all connected subscribers.

---

## 4. Ecosystem & Infrastructure Context

| Dependency / System | Tier / Type | Architectural Role |
| :--- | :--- | :--- |
| **[[entities/order-matching-engine|Order Matching Engine]]** (`astrophage/order-matching-engine`) | Upstream Service | Core matching engine emitting trade ticks and L2 aggregated order books. |
| **[[entities/compliance-surveillance-monitor|Compliance Surveillance Monitor]]** (`astrophage/compliance-surveillance-monitor`) | Sister Service | Downstream audit system ingesting trade feeds and gateway telemetry (`nte.mdg.telemetry`). |
| **[[entities/trade-settlement-system|Trade Settlement System]]** (`astrophage/trade-settlement-system`) | Downstream Service | Post-trade clearing platform providing reference data and settling matched trades. |
| **[[entities/kafka-broker|Kafka Broker]]** (`kafka1.nte.internal:9092`) | Infrastructure | High-throughput distributed message log for exchange events. |
| **[[entities/redis-cache|Redis Cache]]** (`redis:alpine`) | Infrastructure | In-memory key-value cache used for instantaneous L2 snapshot hydration on new client connections. |

---

## 5. Architectural Summaries & References

* High-level architectural index: [[index]]
* Complete message flow and lifecycle: [[concepts/data-lifecycle]]
* Kafka decoupling strategies: [[concepts/ingestion-egress-decoupling]]
* WebSocket fan-out mechanics: [[concepts/websocket-broadcasting]]
* Snapshot hydration architecture: [[concepts/state-hydration]]
* Architectural decision records:
  * [[decisions/independent-consumer-groups]]
  * [[decisions/low-latency-egress-design]]
  * [[decisions/redis-l2-state-caching]]