<!-- anchor: src/consumers/OrderBookConsumer.ts:L1-L100 sha:HEAD -->

# Kafka Broker Infrastructure

The **Kafka Broker** infrastructure serves as the high-throughput, fault-tolerant distributed streaming backbone for the **Nexus Trading Exchange (NTE)** platform. Within the Market Data Gateway (MDG) ecosystem, Apache Kafka decouples internal execution engines from external dissemination layers, providing guaranteed message persistence and partition-level ordering for order book depth snapshots and trade execution ticks.

---

## Responsibilities

* **Ingress Message Ingestion & Buffering**: Ingests and persists canonical market events emitted in real time by the upstream [[entities/order-matching-engine]].
* **Partitioned Stream Ordering**: Guarantees deterministic, sequential event ordering per trading symbol/partition.
* **Consumer Group Isolation**: Enables independent offset tracking and horizontal scaling across isolated consumer pipelines (such as `mdg-orderbook-group` and `mdg-trade-group`).
* **Cross-Service Decoupling**: Shields the matching engine from external consumption latency and WebSocket client connection state, facilitating [[concepts/ingestion-egress-decoupling]].
* **Broadcast Integration**: Provides continuous event feeds to downstream analytics, clearing, and surveillance systems such as the [[entities/compliance-surveillance-monitor]] and [[entities/trade-settlement-system]].

---

## Dependencies

* **Upstream Producers**:
  * `astrophage/[[entities/order-matching-engine]]`: Publishes execution events and Level 2 (L2) book state deltas to internal topics.
* **Downstream Consumers**:
  * `[[entities/order-book-consumer]]`: Subscribes to `nte.orderbook.snapshots` using client ID `mdg-ob-consumer`.
  * `[[entities/trade-consumer]]`: Subscribes to `nte.trades.matched` using client ID `mdg-trade-consumer`.
  * `astrophage/[[entities/compliance-surveillance-monitor]]`: Subscribes to telemetry and trading event streams.
  * `astrophage/[[entities/trade-settlement-system]]`: Ingests trade matches for post-trade clearing and settlement.
* **Wire Serialization**:
  * `[[entities/protobuf-contracts]]` (`protos/market_data.proto`): Defines the Protocol Buffer payloads transmitted across the broker topics.
* **Client Driver**:
  * `kafkajs`: High-performance Node.js Kafka client utilized by MDG consumers.

---

## Topic Specifications & Schemas

The MDG interfaces with core event streams defined by strict [[entities/protobuf-contracts]]:

```
                                 Kafka Topics
 ┌───────────────────────────┐                  ┌──────────────────────────┐
 │  nte.orderbook.snapshots  │                  │    nte.trades.matched    │
 └─────────────┬─────────────┘                  └────────────┬─────────────┘
               │                                             │
               │ (OrderBookSnapshot)                         │ (TradeTick)
               v                                             v
 ┌───────────────────────────┐                  ┌──────────────────────────┐
 │   `OrderBookConsumer`   │                  │     `TradeConsumer`    │
 │ (grp: mdg-orderbook-group)│                  │  (grp: mdg-trade-group)  │
 └───────────────────────────┘                  └──────────────────────────┘
```

### 1. `nte.orderbook.snapshots`
* **Producer**: [[entities/order-matching-engine]]
* **Consumer**: [[entities/order-book-consumer]]
* **Payload Format**: Protobuf `nte.marketdata.OrderBookSnapshot`
* **Description**: Carries periodic and event-driven aggregated Level 2 depth books, containing bids and asks with corresponding price, quantity, and order count fields.
* **Key Fields**:
  * `symbol` (`string`)
  * `bids` (`repeated Level`)
  * `asks` (`repeated Level`)
  * `timestamp_ns` (`int64`)

### 2. `nte.trades.matched`
* **Producer**: [[entities/order-matching-engine]]
* **Consumer**: [[entities/trade-consumer]]
* **Payload Format**: Protobuf `nte.marketdata.TradeTick`
* **Description**: Carries real-time execution records produced whenever orders cross in the matching engine order book.
* **Key Fields**:
  * `trade_id` (`string`)
  * `symbol` (`string`)
  * `price` (`double`)
  * `quantity` (`double`)
  * `matched_at_ns` (`int64`)
  * `taker_side` (`string` — `"BUY"` | `"SELL"`)

### 3. `nte.mdg.telemetry`
* **Producer**: Market Data Gateway telemetry agents
* **Consumer**: [[entities/compliance-surveillance-monitor]]
* **Description**: Disseminates operational metrics, egress latency benchmarks, and broadcast telemetry for audit verification.

---

## Consumer Group Configuration

To maximize throughput and ensure stream isolation, the MDG utilizes separate consumer groups per event stream:

| Consumer Service | Kafka Client ID | Consumer Group ID | Subscribed Topic | `fromBeginning` |
| :--- | :--- | :--- | :--- | :--- |
| `[[entities/order-book-consumer]]` | `mdg-ob-consumer` | `mdg-orderbook-group` | `nte.orderbook.snapshots` | `false` |
| `[[entities/trade-consumer]]` | `mdg-trade-consumer` | `mdg-trade-group` | `nte.trades.matched` | `false` |

### Partitioning & Isolation Strategy
* **Distinct Consumer Groups**: As detailed in [[decisions/independent-consumer-groups]], isolating `OrderBookConsumer` and `TradeConsumer` ensures that partition rebalances, slow deserialization, or offset commits on one topic do not cascade or stall the other.
* **Offset Policy**: Consumers start with `fromBeginning: false` on startup, attaching to latest offsets to prioritize real-time market data over historical replays. Stale order book state is avoided by hydrating from the [[entities/redis-cache]] (see [[concepts/state-hydration]] and [[decisions/redis-l2-state-caching]]).

---

## Network & Connection Topography

In production environments, the broker cluster runs internally under the host address `kafka1.nte.internal:9092`. For local development and containerized orchestration via Docker Compose, connection strings are configurable via environment variables:

```bash
# Environment Configuration
KAFKA_BROKERS=kafka1.nte.internal:9092
ORDERBOOK_TOPIC=nte.orderbook.snapshots
TRADES_TOPIC=nte.trades.matched
```

Each consumer initializes an isolated `KafkaJS` client connection pool, handling automatic heartbeat management, partition rebalancing, and TCP reconnect loops during transient network disruptions. See [[concepts/data-lifecycle]] for the end-to-end data progression through the Kafka broker to [[entities/client-manager]].