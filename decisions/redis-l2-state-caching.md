# ADR: Redis for Rapid L2 Order Book Hydration

## Status
Accepted

## Context
External market participants and trading interfaces connect to the **Nexus Trading Exchange (NTE)** via the [[summaries/market-data-gateway-overview|Market Data Gateway (MDG)]] WebSocket server managed by [[entities/client-manager|ClientManager]]. Upon establishing a new WebSocket connection, a client requires an immediate, canonical Level 2 (L2) depth snapshot (`nte.marketdata.OrderBookSnapshot`) for active trading symbols before it can apply incremental live delta broadcasts (`L2_UPDATE`).

Three architectural approaches were evaluated to provide this initial state during [[concepts/state-hydration|state hydration]]:

1. **Direct Query to Matching Engine**: Directing snapshot requests synchronously back to the upstream [[entities/order-matching-engine|Order Matching Engine]] via RPC.
   * *Drawback*: Introduces significant computational load and network contention to the deterministic matching core, risking latency degradation for execution paths.
2. **Passive Waiting on Kafka Stream**: Forcing connecting clients to wait until the next periodic snapshot record arrives on the `nte.orderbook.snapshots` [[entities/kafka-broker|Kafka]] topic via the [[entities/order-book-consumer|OrderBookConsumer]].
   * *Drawback*: Produces non-deterministic onboarding latency for client interfaces and creates potential race conditions where clients miss intermediate price updates.
3. **Local In-Memory Process Caching Only**: Storing the latest snapshot purely in Node.js process memory within each MDG instance.
   * *Drawback*: Breaks down during rolling restarts, cold starts, and horizontal scaling of MDG instances where newly spawned edge pods lack historical state prior to consuming from Kafka.

## Decision
We decided to integrate **Redis** (`[[entities/redis-cache]]`) as an out-of-process, high-throughput in-memory state store to maintain the latest L2 order book snapshot per trading symbol.

```
+-----------------------------+
|    `OrderBookConsumer`    |
+-----------------------------+
       |                  \
       | (Stream Update)   \ (Write Latest L2 Snapshot)
       v                    v
+------------------+    +-------------------+
| `ClientManager`|    |   `Redis Cache`  |
+------------------+    +-------------------+
       ^                          |
       | (Read Snapshot on Open)  |
       +--------------------------+
       |
       v
+------------------+
|    WS Client     |
| (Hydrated State) |
+------------------+
```

### Key Implementation Facets:
1. **Asynchronous Snapshot Persistence**: As the [[entities/order-book-consumer|OrderBookConsumer]] ingests snapshots from the `nte.orderbook.snapshots` topic (defined in [[entities/protobuf-contracts|market_data.proto]]), it updates the latest snapshot representation in [[entities/redis-cache|Redis]] using key patterns structured by market symbol (e.g., `nte:l2:snapshot:{symbol}`).
2. **Instantaneous Client Hydration**: During the WebSocket connection handshake in [[entities/client-manager|ClientManager]], MDG executes an asynchronous lookup against [[entities/redis-cache|Redis]] to retrieve the current L2 state and immediately dispatches a hydrated snapshot to the client.
3. **Streaming Decoupling**: Once hydrated, the client socket is enrolled into the active connection pool for live fan-out broadcasting via [[concepts/websocket-broadcasting|WebSocket Broadcasting]] and [[concepts/ingestion-egress-decoupling|Ingestion-Egress Decoupling]].

## Consequences

### Positive
* **Sub-Millisecond Onboarding**: Reduces client [[concepts/state-hydration|state hydration]] time to low-millisecond/sub-millisecond Redis `GET` operations.
* **Core Engine Isolation**: Fully protects the [[entities/order-matching-engine|Order Matching Engine]] from client-driven query storms during volatile market events or exchange reconnections.
* **Resilience Across Restarts**: Newly deployed MDG instances can immediately serve initial L2 snapshots to clients without requiring topic re-reads from offset zero on [[entities/kafka-broker|Kafka]].
* **Consistent State Representation**: Provides a single source of truth for the latest market depth across distributed MDG edge replicas.

### Negative / Trade-offs
* **Infrastructure Dependency**: Introduces Redis as an essential component within `docker-compose.yml` and production infrastructure alongside Kafka.
* **Data Serialization Overhead**: Requires continuous serialization/deserialization between [[entities/protobuf-contracts|Protobuf]] wire messages, Redis key-value strings, and [[entities/runtime-models|runtime JSON types]].
* **Timestamp Alignment Management**: MDG must ensure clients do not apply stale Kafka delta updates that precede the `timestamp_ns` of the hydrated Redis snapshot.

## Related Artifacts
* [[index]]
* [[summaries/market-data-gateway-overview]]
* [[concepts/data-lifecycle]]
* [[concepts/state-hydration]]
* [[concepts/websocket-broadcasting]]
* [[entities/redis-cache]]
* [[entities/order-book-consumer]]
* [[entities/client-manager]]
* [[decisions/low-latency-egress-design]]
* [[decisions/independent-consumer-groups]]