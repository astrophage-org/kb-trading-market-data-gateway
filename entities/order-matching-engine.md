<!-- anchor: docs/architecture.md:L1-L100 sha:HEAD -->

# Order Matching Engine

The **Order Matching Engine** (`astrophage/order-matching-engine`) is the central core execution tier within the **Global Financial Markets Group (GFMG) Nexus Trading Exchange (NTE)** platform. It processes incoming participant orders, manages matching algorithms (such as continuous price-time priority crossing), executes trades, and aggregates internal book state.

The matching engine serves as the canonical upstream source of market events for the [[summaries/market-data-gateway-overview|Market Data Gateway (MDG)]], publishing execution records and market depth streams to the [[entities/kafka-broker|Apache Kafka broker]].

---

## Responsibilities

The primary responsibilities of the Order Matching Engine in relation to market data generation include:

* **Order Execution and Crossing**: Continuously matching resting limit orders against incoming aggressive orders across supported trading pairs.
* **Level 2 (L2) Depth Aggregation**: Constructing deterministic aggregated price-level depth books (bids and asks with price, total quantity, and order count) from the internal Level 3 order state.
* **Trade Match Emission**: Generating canonical trade confirmation events and publishing them directly to the `nte.trades.matched` Kafka topic.
* **Order Book Snapshot Generation**: Emitting regular and delta-triggered L2 order book depth snapshots to the `nte.orderbook.snapshots` Kafka topic.
* **Wire Protocol Compliance**: Encoding market events according to the shared [[entities/protobuf-contracts|Protobuf Wire Contracts]] (`market_data.proto`) to maintain contract consistency across downstream consumers.

---

## Dependencies

The Order Matching Engine integrates across several critical exchange subsystems and downstream consumers:

### Messaging Infrastructure
* **[[entities/kafka-broker|Kafka Broker]]**: Transports raw event streams produced by the matching engine across distinct topics (`nte.orderbook.snapshots` and `nte.trades.matched`).

### Upstream/Downstream Services
* **[[summaries/market-data-gateway-overview|Market Data Gateway (MDG)]]**:
  * **[[entities/order-book-consumer|OrderBookConsumer]]**: Ingests L2 snapshots for client broadcast and [[concepts/state-hydration|state hydration]].
  * **[[entities/trade-consumer|TradeConsumer]]**: Ingests execution ticks for real-time WebSocket client distribution.
* **[[entities/compliance-surveillance-monitor|Compliance Surveillance Monitor]]**: Downstream sister service ingesting trade and order events for regulatory audit and market abuse surveillance.
* **[[entities/trade-settlement-system|Trade Settlement System]]**: Downstream service ingesting confirmed trade executions for post-trade clearing and clearinghouse allocation.

### Contract Layer
* **[[entities/protobuf-contracts|Protobuf Contracts (`market_data.proto`)]]**: Common schemas governing message structure between the matching core and ingestion services.

---

## Event Production & Schemas

The matching engine emits two primary event streams ingested by the MDG:

```
+-----------------------------------------------------------------+
|                   Order Matching Engine Core                    |
+-----------------------------------------------------------------+
               |                                     |
               | (Aggregated Book Depths)            | (Execution Matches)
               v                                     v
+-----------------------------+       +-----------------------------+
|    nte.orderbook.snapshots  |       |     nte.trades.matched      |
|  (nte.marketdata.           |       |  (nte.marketdata.           |
|   OrderBookSnapshot)        |       |   TradeTick)                |
+-----------------------------+       +-----------------------------+
```

### 1. Order Book Snapshots (`nte.orderbook.snapshots`)
Represents the current bid and ask depth distribution for a given instrument.

* **Protobuf Schema**: `nte.marketdata.OrderBookSnapshot`
* **Payload Fields**:
  * `symbol` (`string`): Ticker identifier (e.g., `BTC-USD`).
  * `bids` (`repeated Level`): List of aggregated price levels on the buy side.
  * `asks` (`repeated Level`): List of aggregated price levels on the sell side.
  * `timestamp_ns` (`int64`): High-precision UTC timestamp (nanoseconds) at book generation.
* **Level Structure** (`nte.marketdata.Level`):
  * `price` (`double`): Aggregated limit price.
  * `quantity` (`double`): Aggregate volume available at this price.
  * `order_count` (`int32`): Total number of discrete limit orders at this level.

### 2. Matched Trades (`nte.trades.matched`)
Represents an instantaneous crossing of a resting order and an incoming taker order.

* **Protobuf Schema**: `nte.marketdata.TradeTick`
* **Payload Fields**:
  * `trade_id` (`string`): Unique trade identifier assigned by the engine.
  * `symbol` (`string`): Ticker identifier.
  * `price` (`double`): Executed price.
  * `quantity` (`double`): Executed volume.
  * `matched_at_ns` (`int64`): Nanosecond execution timestamp.
  * `taker_side` (`string`): Side of the aggressive order (`BUY` or `SELL`).

---

## Integration with Market Data Gateway

1. **Decoupled Egress**: The matching engine writes to Kafka asynchronously, isolating engine core performance from edge delivery latencies (see [[concepts/ingestion-egress-decoupling|Ingestion & Egress Decoupling]]).
2. **Stream Parallelism**: Events are partitioned and consumed independently by the [[entities/order-book-consumer|OrderBookConsumer]] and [[entities/trade-consumer|TradeConsumer]] using isolated consumer groups (see [[decisions/independent-consumer-groups|Independent Consumer Groups]]).
3. **Downstream Mapping**:
   * Order book snapshots are translated to `L2_UPDATE` envelopes and cached in [[entities/redis-cache|Redis]] for [[decisions/redis-l2-state-caching|L2 State Caching]].
   * Trade ticks are mapped to `TRADE_TICK` envelopes and fanned out immediately via the [[entities/client-manager|ClientManager]] (see [[decisions/low-latency-egress-design|Low-Latency Egress Design]] and [[concepts/websocket-broadcasting|WebSocket Broadcasting]]).
4. **End-to-End Tracing**: The execution timestamp (`timestamp_ns` / `matched_at_ns`) generated by the engine is preserved through the [[concepts/data-lifecycle|Data Lifecycle]] and mapped into [[entities/runtime-models|runtime types]] to enable end-to-end latency measurement at client endpoints.