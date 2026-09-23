# Architectural Summary: Market Data Gateway (MDG)

The **Market Data Gateway (MDG)** (`@astrophage/market-data-gateway`) is a high-performance edge service within the **Global Financial Markets Group (GFMG) Nexus Trading Exchange (NTE)** platform. It functions as the primary real-time egress tier, ingesting order book depth updates and execution ticks from the internal matching core and fanning them out to external market participants and client interfaces via ultra-low-latency WebSockets.

---

## 1. System Topology & Architecture

The MDG decouples internal message broker topologies from public distribution protocols, acting as an event-driven bridge between Apache Kafka and client WebSockets.

```
                   +------------------------------------+
                   |     `Order Matching Engine`      |
                   +------------------------------------+
                                      |
              +-----------------------+-----------------------+
              | (nte.orderbook.snapshots)                     | (nte.trades.matched)
              v                                               v
+-----------------------------+               +-----------------------------+
|    `OrderBookConsumer`    |               |      `TradeConsumer`      |
|  (grp: mdg-orderbook-group) |               |    (grp: mdg-trade-group)   |
+-----------------------------+               +-----------------------------+
              \                                               /
               \                                             /
                v                                           v
      +---------------------------------------------------------------+
      |                      `ClientManager`                        |
      |          (WebSocket Connection Pool & Fan-out Engine)         |
      +---------------------------------------------------------------+
                                      |
                    (JSON / L2_UPDATE & TRADE_TICK)
                                      v
                      +-------------------------------+
                      | External Clients / UI Egress  |
                      +-------------------------------+
```

---

## 2. Core Components

The application is structured into four primary modules:

### 2.1. Bootstrap & Runtime Orchestration (`src/index.ts`)
* Acts as the application entry point.
* Initializes the `ClientManager` WebSocket server on TCP port `8080`.
* Instantiates and connects the Kafka stream consumers.
* Handles graceful startup, lifecycle coordination, and process termination signals.

### 2.2. Ingestion Subsystem (`src/consumers/`)
The ingestion tier runs asynchronous Kafka consumers to process upstream trading events:
* **``OrderBookConsumer``**: Subscribes to the `nte.orderbook.snapshots` topic under consumer group `mdg-orderbook-group`. Handles aggregated Level 2 (L2) depth snapshots.
* **``TradeConsumer``**: Subscribes to the `nte.trades.matched` topic under consumer group `mdg-trade-group`. Ingests confirmed trade match ticks.

### 2.3. Dissemination & Connection Subsystem (`src/websockets/ClientManager.ts`)
* Encapsulates WebSocket server lifecycle and active connection pooling using the `ws` library.
* Maintains an in-memory registry of active subscriber sockets.
* Implements high-throughput non-blocking broadcast dispatching:
  * `broadcastOrderBook(data)`: Emits `{ type: "L2_UPDATE", data: ... }` payloads.
  * `broadcastTrade(data)`: Emits `{ type: "TRADE_TICK", data: ... }` payloads.
* Validates socket connection states (`WebSocket.OPEN`) prior to write operations to prevent backpressure stalls and drop disconnected clients.

### 2.4. Contract & Data Modeling Layer (`protos/` & `src/models/`)
* **Protobuf Wire Contracts (`protos/market_data.proto`)**: Defines the cross-service schema shared between the `order-matching-engine`, `compliance-surveillance-monitor`, and MDG:
  * `nte.marketdata.OrderBookSnapshot`: Contains `symbol`, `timestamp_ns`, and repeated `Level` messages (`price`, `quantity`, `order_count`).
  * `nte.marketdata.TradeTick`: Contains `trade_id`, `symbol`, `price`, `quantity`, `matched_at_ns`, and `taker_side`.
* **TypeScript Runtime Types (`src/models/types.ts`)**: Defines strongly typed runtime data models (`OrderBookEvent`, `TradeEvent`, `PriceLevel`).

---

## 3. End-to-End Data Lifecycle

```
[Matching Core Engine]
        │
        ├──► Kafka: `nte.orderbook.snapshots` ──► `OrderBookConsumer` ──► `ClientManager.broadcastOrderBook` ──► WS Clients (L2_UPDATE)
        │
        └──► Kafka: `nte.trades.matched`      ──► `TradeConsumer`     ──► `ClientManager.broadcastTrade`     ──► WS Clients (TRADE_TICK)
```

1. **Generation**: The upstream matching engine executes trades and aggregates resting limit orders into discrete L2 depth snapshots.
2. **Ingestion**: MDG consumers asynchronously pull records from their respective Kafka partitions.
3. **Envelope Packaging**: Ingested payloads are formatted into standardized JSON event envelopes (`L2_UPDATE` or `TRADE_TICK`).
4. **Fan-out Broadcast**: The ``ClientManager`` iterates over active client sockets and broadcasts the messages.

---

## 4. Key Architectural & Design Decisions

* **Independent Consumer Groups**: ``OrderBookConsumer`` and ``TradeConsumer`` operate under distinct consumer group IDs (`mdg-orderbook-group` and `mdg-trade-group`). This prevents partition rebalances in one stream from stalling the other and permits independent horizontal scaling.
* **Ingestion/Egress Decoupling**: Kafka ingestion runs asynchronously from client distribution, insulating the core broker cluster from slow or high-latency external WebSocket connections.
* **Low-Latency Egress Path**: The broadcast loop executes fast in-memory iterations over open sockets with minimal transformation overhead.
* **State Caching (Redis)**: Redis is integrated into the infrastructure tier to store the latest L2 book state, allowing rapid state hydration for newly connected clients during connection handshakes.

---

## 5. Ecosystem & Infrastructure Context

| Dependency / System | Type | Role |
| :--- | :--- | :--- |
| **`astrophage/order-matching-engine`** | Upstream Service | Generates canonical trade executions and L2 depth updates. |
| **`astrophage/compliance-surveillance-monitor`** | Sister Service | Downstream consumer of market data feeds for real-time audit and surveillance. |
| **`astrophage/trade-settlement-system`** | Downstream Service | Ingests matched trades for post-trade clearing and settlement. |
| **Apache Kafka** (`kafka1.nte.internal:9092`) | Infrastructure | High-throughput distributed message bus for market events. |
| **Redis** | Infrastructure | In-memory key-value cache for instantaneous L2 snapshot retrieval. |