<!-- anchor: docs/architecture.md:L1-L100 sha:HEAD -->

# OrderBookConsumer

The `OrderBookConsumer` is an ingestion service within the **Market Data Gateway (MDG)** responsible for consuming aggregated Level 2 (L2) order book snapshots published by the [[entities/order-matching-engine]]. It acts as the intake pipeline for market depth, processing incoming Kafka messages and passing them directly to the [[entities/client-manager]] for non-blocking distribution across active WebSocket connections.

Defined in `src/consumers/OrderBookConsumer.ts`, the consumer encapsulates Apache Kafka subscription logic and decouples broker stream consumption from downstream client communication.

---

## ## Responsibilities

* **Kafka Stream Ingestion**: Connect to the [[entities/kafka-broker]] cluster and subscribe to the `nte.orderbook.snapshots` topic.
* **Consumer Group Isolation**: Maintain an independent Kafka consumer group (`mdg-orderbook-group`) to isolate order book partition rebalancing and offset commits from the trade ingestion pipeline handled by [[entities/trade-consumer]] (see [[decisions/independent-consumer-groups]]).
* **Payload Ingestion & Deserialization**: Ingest raw byte payloads conforming to the Protobuf contract `nte.marketdata.OrderBookSnapshot` defined in [[entities/protobuf-contracts]].
* **Egress Handoff**: Forward ingested snapshot payloads to the [[entities/client-manager]] instance via `broadcastOrderBook()`, triggering an `L2_UPDATE` fan-out broadcast (see [[concepts/websocket-broadcasting]] and [[entities/runtime-models]]).
* **Broker Decoupling**: Enable [[concepts/ingestion-egress-decoupling]] by absorbing high-throughput L2 depth batches without being blocked by client-side network latency or backpressure.

---

## ## Dependencies

* **[[entities/kafka-broker]]**: Requires access to Kafka brokers (configured via `process.env.KAFKA_BROKERS`, defaulting to `kafka1.nte.internal:9092` / `localhost:9092`).
* **[[entities/order-matching-engine]]**: Upstream producer publishing periodic aggregated order book depth frames to `nte.orderbook.snapshots`.
* **[[entities/client-manager]]**: Injected via constructor (`ClientManager`); consumes processed snapshot payloads for WebSocket broadcasting.
* **[[entities/protobuf-contracts]]**: Implements decoding against the `nte.marketdata.OrderBookSnapshot` wire schema.
* **[[entities/redis-cache]]**: Interacts with L2 book state storage to facilitate [[concepts/state-hydration]] for new client handshakes (see [[decisions/redis-l2-state-caching]]).
* **`kafkajs`**: Underlying driver library for managing Kafka connections, consumer heartbeats, and partition assignment.

---

## Component Configuration & Lifecycle

The consumer is instantiated during application bootstrap in `src/index.ts` alongside the [[entities/trade-consumer]] and [[entities/client-manager]]:

```typescript
// src/consumers/OrderBookConsumer.ts
import { Kafka } from 'kafkajs';
import { ClientManager } from '../websockets/ClientManager';

export class OrderBookConsumer {
    private kafka = new Kafka({ 
        clientId: 'mdg-ob-consumer', 
        brokers: [process.env.KAFKA_BROKERS || 'localhost:9092'] 
    });
    private consumer = this.kafka.consumer({ groupId: 'mdg-orderbook-group' });

    constructor(private clientManager: ClientManager) {}

    async start() {
        await this.consumer.connect();
        await this.consumer.subscribe({ 
            topic: 'nte.orderbook.snapshots', 
            fromBeginning: false 
        });

        await this.consumer.run({
            eachMessage: async ({ topic, partition, message }) => {
                const snapshot = message.value?.toString(); 
                if (snapshot) {
                    this.clientManager.broadcastOrderBook(snapshot);
                }
            }
        });
    }
}
```

### Kafka Configuration Parameters

| Parameter | Value | Description |
| :--- | :--- | :--- |
| `clientId` | `mdg-ob-consumer` | Identifier assigned to the Kafka client connection. |
| `groupId` | `mdg-orderbook-group` | Dedicated consumer group ID isolating depth stream processing. |
| `topic` | `nte.orderbook.snapshots` | Topic containing L2 snapshot streams. |
| `fromBeginning` | `false` | Subscribes only to real-time events emitted after consumer attachment. |

---

## Data Ingestion & Transformation Flow

The position of `OrderBookConsumer` within the [[concepts/data-lifecycle]] is illustrated below:

```
+--------------------------------+
| `Order Matching Engine`      |
+--------------------------------+
               |
               | (nte.orderbook.snapshots)
               v
+--------------------------------+
| `OrderBookConsumer`          |
| Client ID: mdg-ob-consumer     |
| Group: mdg-orderbook-group     |
+--------------------------------+
               |
               | clientManager.broadcastOrderBook(snapshot)
               v
+--------------------------------+
| `ClientManager`              |
+--------------------------------+
               |
               | JSON Envelope: { type: "L2_UPDATE", data: ... }
               v
+--------------------------------+
| Connected WebSocket Clients    |
+--------------------------------+
```

1. **Ingest**: The consumer receives a Kafka message batch from its assigned partition.
2. **Decode**: The binary message value is extracted. In production pipelines, this maps to the `OrderBookSnapshot` Protobuf structure:
   - `symbol` (`string`)
   - `bids` (`repeated Level` containing `price`, `quantity`, `order_count`)
   - `asks` (`repeated Level` containing `price`, `quantity`, `order_count`)
   - `timestamp_ns` (`int64`)
3. **Dispatch**: The payload is passed to `clientManager.broadcastOrderBook()`, wrapping the data into an `L2_UPDATE` runtime envelope (as modeled in [[entities/runtime-models]]) and dispatching to all open WebSocket sessions via [[concepts/websocket-broadcasting]].

---

## Related Documentation

* **Overview**: [[summaries/market-data-gateway-overview]]
* **Trade Ingestion Counterpart**: [[entities/trade-consumer]]
* **Client Fan-Out Engine**: [[entities/client-manager]]
* **Architecture Decisions**: [[decisions/independent-consumer-groups]], [[decisions/redis-l2-state-caching]]
* **Data Flow & Decoupling**: [[concepts/data-lifecycle]], [[concepts/ingestion-egress-decoupling]]