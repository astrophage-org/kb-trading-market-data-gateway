# Architectural Decision Record: Independent Consumer Groups for OrderBook and Trade Streams

## Status
**Accepted** (Implemented in `v1.0.0`)

---

## Context
The Market Data Gateway (MDG) consumes two primary market data event streams published to the [[entities/kafka-broker]] by the upstream [[entities/order-matching-engine]]:
1. `nte.orderbook.snapshots`: Aggregated Level 2 (L2) depth snapshots containing arrays of bids, asks, quantities, and order counts defined by [[entities/protobuf-contracts]].
2. `nte.trades.matched`: High-frequency execution ticks containing trade identifiers, execution prices, quantities, and taker sides.

In high-throughput trading environments, these two data streams exhibit vastly different operational and performance characteristics:
* **Payload Size & Processing Overhead**: Order book snapshots contain complex arrays of depth levels (`nte.marketdata.OrderBookSnapshot`), resulting in larger payload sizes and higher deserialization costs. Trade ticks (`nte.marketdata.TradeTick`) are compact, lightweight discrete events.
* **Throughput & Burst Characteristics**: Order book snapshot generation frequency varies with quote replacement rates and book depth changes, while trade tick volume spikes during active fills and aggressive market orders.
* **Latency Sensitivity**: Executed trade events require immediate dissemination to external clients to ensure prompt trade tape updates and fair execution reporting.

In early architectural reviews, using a single unified Kafka consumer group subscribing to both topics (`nte.orderbook.snapshots` and `nte.trades.matched`) was evaluated. However, this approach presented critical operational failure modes:
* **Head-of-Line Blocking**: Slower deserialization or processing of bulky order book snapshots on a shared consumer thread would stall the delivery of real-time trade ticks.
* **Coupled Partition Rebalances**: In a unified consumer group, a partition rebalance caused by dynamic worker scaling, broker node failover, or transient heartbeat timeouts would pause consumption across *both* trade and order book streams simultaneously.
* **Scaling Inelasticity**: Inability to independently scale partition allocations and consumer instances based on the specific volume characteristics of trades versus depth snapshots.

---

## Decision
We decided to completely decouple the ingestion pipelines by instantiating two independent Kafka consumer groups:

1. **Order Book Stream Consumer**:
   * **Component**: [[entities/order-book-consumer]] (`src/consumers/OrderBookConsumer.ts`)
   * **Topic**: `nte.orderbook.snapshots`
   * **Consumer Group ID**: `mdg-orderbook-group`
   * **Client ID**: `mdg-ob-consumer`

2. **Trade Stream Consumer**:
   * **Component**: [[entities/trade-consumer]] (`src/consumers/TradeConsumer.ts`)
   * **Topic**: `nte.trades.matched`
   * **Consumer Group ID**: `mdg-trade-group`
   * **Client ID**: `mdg-trade-consumer`

```
                      +------------------------------------+
                      |     [[entities/order-matching-engine]]      |
                      +------------------------------------+
                                         |
                 +-----------------------+-----------------------+
                 | (nte.orderbook.snapshots)                     | (nte.trades.matched)
                 v                                               v
+--------------------------------+               +--------------------------------+
|    [[entities/order-book-consumer]]    |               |      [[entities/trade-consumer]]      |
|   (grp: mdg-orderbook-group)   |               |     (grp: mdg-trade-group)     |
+--------------------------------+               +--------------------------------+
                 \                                               /
                  \                                             /
                   v                                           v
         +---------------------------------------------------------------+
         |                      [[entities/client-manager]]                      |
         |          (WebSocket Connection Pool & Fan-out Engine)         |
         +---------------------------------------------------------------+
```

Both consumers operate independently within the runtime process orchestrated by `src/index.ts`, passing unpacked payloads directly to the shared [[entities/client-manager]] for client fan-out broadcasting (`[[concepts/websocket-broadcasting]]`).

---

## Consequences

### Positive
* **Fault & Rebalance Isolation**: Partition rebalances or consumer group adjustments on `mdg-orderbook-group` have zero impact on `mdg-trade-group`. Real-time trade tick delivery continues uninterrupted during depth stream rebalances.
* **Zero Head-of-Line Contention**: Trade processing loops run asynchronously from depth snapshot processing, maintaining ultra-low latency for trade tick dissemination (`TRADE_TICK` event envelopes in [[entities/runtime-models]]).
* **Independent Scalability & Tuning**: Consumer concurrency, batch sizes, fetch buffer sizes, and Kafka partition counts can be tuned and scaled independently for trade executions and book snapshots.
* **Architectural Decoupling**: Reinforces [[concepts/ingestion-egress-decoupling]], allowing stream-specific logic (such as caching L2 snapshots in [[entities/redis-cache]] per [[decisions/redis-l2-state-caching]]) without polluting the trade egress path.

### Negative / Trade-offs
* **Resource Overhead**: Requires maintaining two separate Kafka consumer instances, TCP connections, and polling loops per MDG instance, slightly increasing socket and memory overhead.
* **Orchestration Complexity**: Application lifecycle management (`src/index.ts`) must coordinate the startup, health check monitoring, and graceful shutdown of multiple consumer groups.

---

## Related Documentation
* [[summaries/market-data-gateway-overview]]
* [[concepts/data-lifecycle]]
* [[concepts/ingestion-egress-decoupling]]
* [[entities/order-book-consumer]]
* [[entities/trade-consumer]]
* [[entities/kafka-broker]]
* [[decisions/low-latency-egress-design]]
* [[decisions/redis-l2-state-caching]]