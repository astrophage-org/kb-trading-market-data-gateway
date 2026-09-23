# End-to-End Data Lifecycle

The **Data Lifecycle** within the Market Data Gateway (MDG) describes the path market data takes from its initial generation in the internal matching core to its eventual distribution across external WebSocket client connections. The architecture guarantees deterministic, low-latency market data delivery while maintaining strict separation between internal broker topology and public distribution networks.

For a broader perspective on MDG topology and components, see [[summaries/market-data-gateway-overview]] and the [[index]].

---

## Lifecycle Pipeline Overview

The market data propagation path consists of five discrete stages across internal and edge tiers:

```
+---------------------------------------------------------------------------------------------------+
| 1. GENERATION & UPSTREAM PUBLISHING                                                               |
|    [[entities/order-matching-engine]]                                                             |
|    - Matches limit orders & generates trades                                                      |
|    - Aggregates resting orders into Level 2 depth books                                           |
|    - Serializes via [[entities/protobuf-contracts]]                                               |
+---------------------------------------------------------------------------------------------------+
                                         │
                                         ▼
+---------------------------------------------------------------------------------------------------+
| 2. DISTRIBUTED MESSAGE TRANSPORT                                                                  |
|    [[entities/kafka-broker]]                                                                      |
|    - Topic: `nte.orderbook.snapshots` (Aggregated Depth)                                          |
|    - Topic: `nte.trades.matched` (Execution Ticks)                                                |
|    - Parallel feeds consumed by [[entities/compliance-surveillance-monitor]] & [[entities/trade-settlement-system]] |
+---------------------------------------------------------------------------------------------------+
                                         │
                                         ▼
+---------------------------------------------------------------------------------------------------+
| 3. INGESTION & ISOLATION                                                                          |
|    [[concepts/ingestion-egress-decoupling]]                                                       |
|    - [[entities/order-book-consumer]] (Consumer Group: `mdg-orderbook-group`)                     |
|    - [[entities/trade-consumer]]     (Consumer Group: `mdg-trade-group`)                          |
|    - Isolated stream processing via [[decisions/independent-consumer-groups]]                     |
+---------------------------------------------------------------------------------------------------+
                                         │
                                         ├──────────────────────────┐
                                         │                          │ (Snapshot Cache)
                                         ▼                          ▼
+----------------------------------------------------+    +-----------------------------------------+
| 4. ENVELOPE PACKAGING & MAPPING                    |    | [[concepts/state-hydration]]            |
|    [[entities/runtime-models]]                     |    | [[entities/redis-cache]]                |
|    - `OrderBookEvent` -> `{ type: "L2_UPDATE" }`   |    | - Caches latest L2 state                |
|    - `TradeEvent`     -> `{ type: "TRADE_TICK" }`  |    | - Rapid sync for newly joined clients   |
+----------------------------------------------------+    |   via [[decisions/redis-l2-state-caching]] |
                         │                                +-----------------------------------------+
                         ▼
+---------------------------------------------------------------------------------------------------+
| 5. FAN-OUT EGRESS & CLIENT DISPATCH                                                               |
|    [[entities/client-manager]] & [[concepts/websocket-broadcasting]]                              |
|    - Validates `client.readyState === WebSocket.OPEN`                                             |
|    - Executes non-blocking broadcast dispatching                                                  |
|    - Direct JSON wire streaming via [[decisions/low-latency-egress-design]]                        |
+---------------------------------------------------------------------------------------------------+
                                         │
                                         ▼
                               [ External WS Clients ]
```

---

## Detailed Stage Breakdown

### Stage 1: Generation & Upstream Publishing

Market data originates within the upstream [[entities/order-matching-engine]]. When incoming limit orders execute or adjust resting book liquidity:
1. **Trade Executions**: A trade match generates a trade execution record containing execution price, aggregate quantity, matched timestamp in nanoseconds (`matched_at_ns`), and the aggressor side (`taker_side`).
2. **Order Book Depth**: Aggregate resting order quantities across discrete price levels are compiled into discrete Level 2 order book updates.
3. **Protobuf Encoding**: Events are structured in accordance with canonical schema definitions in [[entities/protobuf-contracts]] (`OrderBookSnapshot` and `TradeTick`).

### Stage 2: Message Broker Routing

The matching engine publishes events across partitioned topics hosted on the [[entities/kafka-broker]]:
* **`nte.orderbook.snapshots`**: Carries structured depth books with bid/ask arrays and cumulative order counts.
* **`nte.trades.matched`**: Carries executed trade records.

These topics also feed parallel downstream systems across the Nexus Trading Exchange (NTE) platform:
* [[entities/compliance-surveillance-monitor]] consumes trade and book feeds for market abuse analysis and regulatory audit trails.
* [[entities/trade-settlement-system]] processes execution logs for post-trade clearing and end-of-day reconciliation.

### Stage 3: Ingestion Tier & Stream Decoupling

Within the MDG runtime, two dedicated consumers subscribe to the broker topics:
* **[[entities/order-book-consumer]]**: Ingests `nte.orderbook.snapshots` under consumer group `mdg-orderbook-group`.
* **[[entities/trade-consumer]]**: Ingests `nte.trades.matched` under consumer group `mdg-trade-group`.

Key architectural properties applied at this stage include:
* **Consumer Group Isolation**: By utilizing independent consumer groups, rebalance events or partition lag on trade consumption cannot degrade order book delivery rates (see [[decisions/independent-consumer-groups]]).
* **Asynchronous Buffer Decoupling**: Kafka ingestion loops run independently from client socket writes, preventing downstream client network latency from causing upstream Kafka partition backpressure (see [[concepts/ingestion-egress-decoupling]]).

### Stage 4: State Hydration & In-Memory Caching

To support seamless client onboarding and crash recovery:
1. When L2 snapshots arrive, the latest order book state is synchronized with [[entities/redis-cache]].
2. When a new client establishes a WebSocket connection, the [[entities/client-manager]] hydrates the socket with the cached L2 state from Redis before subscribing the client to real-time delta broadcasts (see [[concepts/state-hydration]] and [[decisions/redis-l2-state-caching]]).

### Stage 5: Envelope Packaging & Fan-out Dispatch

Upon message receipt from Kafka:
1. **Model Normalization**: The payload is processed against [[entities/runtime-models]] (`OrderBookEvent` and `TradeEvent`).
2. **Enveloping**: The message is packaged into an outbound JSON envelope containing a discriminating `type` tag:
   * **Order Book Event**: `{ type: "L2_UPDATE", data: <payload> }`
   * **Trade Event**: `{ type: "TRADE_TICK", data: <payload> }`
3. **Broadcast Fan-Out**: The [[entities/client-manager]] executes non-blocking broadcast dispatchers (`broadcastOrderBook` or `broadcastTrade`).
4. **Socket State Validation**: Each active socket in the connection pool is checked for `WebSocket.OPEN` status before writing. Inactive or broken sockets are purged immediately to prevent resource leakage (see [[concepts/websocket-broadcasting]]).
5. **Ultra-Low Latency Delivery**: By minimizing in-memory transformations and relying on pre-serialized JSON buffers, MDG achieves sub-millisecond client fan-out (see [[decisions/low-latency-egress-design]]).

---

## Data Transformation Reference

| Stage | Input Representation | Process / Transformation | Output Representation | Destination |
| :--- | :--- | :--- | :--- | :--- |
| **Engine -> Kafka** | Matching core state | Protobuf binary serialization (`market_data.proto`) | `nte.marketdata.OrderBookSnapshot`, `nte.marketdata.TradeTick` | [[entities/kafka-broker]] |
| **Kafka -> MDG** | Kafka binary wire records | `kafkajs` topic partition read | String/buffer representation | [[entities/order-book-consumer]], [[entities/trade-consumer]] |
| **MDG -> Cache** | Deserialized L2 state | Key-value store write | Stored JSON / Binary Snapshot | [[entities/redis-cache]] |
| **MDG -> Egress** | Consumer string payload | JSON envelope construction | `{"type":"L2_UPDATE", ...}`, `{"type":"TRADE_TICK", ...}` | [[entities/client-manager]] |
| **Egress -> Client** | JSON Envelope | Fast in-memory socket transmission | Client WebSocket Frame (TCP) | External Client Terminals |

---

## Related Documentation
* [[concepts/ingestion-egress-decoupling]] — In-depth architectural rationale for Kafka-to-WebSocket separation.
* [[concepts/websocket-broadcasting]] — Implementation mechanics of connection pooling and message dispatch.
* [[concepts/state-hydration]] — Mechanics of initial snapshot hydration for newly connected clients.
* [[decisions/low-latency-egress-design]] — Design considerations for high-throughput broadcast distribution.