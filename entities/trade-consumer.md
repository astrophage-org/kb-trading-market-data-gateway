<!-- anchor: docs/architecture.md:L1-L100 sha:HEAD -->

The **`TradeConsumer`** (`src/consumers/TradeConsumer.ts`) is a core ingestion worker within the [[summaries/market-data-gateway-overview|Market Data Gateway (MDG)]]. It is responsible for consuming real-time trade execution events published by the upstream [[entities/order-matching-engine|Order Matching Engine]] over Apache Kafka and forwarding them to the [[entities/client-manager|ClientManager]] for fan-out over public WebSockets.

---

## Responsibilities

* **Kafka Stream Consumption**: Connects to the [[entities/kafka-broker|Kafka broker cluster]] and subscribes to the `nte.trades.matched` topic.
* **Consumer Group Isolation**: Operates under the dedicated consumer group `mdg-trade-group` to guarantee that trade ingestion throughput and partition rebalancing remain completely isolated from order book depth updates (see [[decisions/independent-consumer-groups|Decision: Independent Consumer Groups]]).
* **Real-Time Offset Management**: Subscribes with `fromBeginning: false` to process live trade executions without replaying historical ticks on gateway restarts.
* **Egress Handoff**: Extracts matched trade payloads and immediately hands them off to [[entities/client-manager|ClientManager.broadcastTrade()]] for packaging into `TRADE_TICK` JSON envelopes and broadcasting to external WebSocket subscribers.
* **Ingestion-Egress Decoupling**: Implements the ingestion side of the [[concepts/ingestion-egress-decoupling|Ingestion/Egress Decoupling]] pattern, keeping broker polling unblocked by slow downstream client sockets.

---

## Dependencies

* **Upstream Infrastructure & Services**:
  * **[[entities/order-matching-engine|Order Matching Engine]]**: Upstream producer of matched execution records.
  * **[[entities/kafka-broker|Kafka Broker]]**: High-throughput message bus exposing the `nte.trades.matched` topic.
* **Internal Application Components**:
  * **[[entities/client-manager|ClientManager]]**: Downstream WebSocket connection pool and broadcasting engine (`broadcastTrade`).
  * **[[entities/protobuf-contracts|Protobuf Contracts]]**: Schema definition for `nte.marketdata.TradeTick` defined in `market_data.proto`.
  * **[[entities/runtime-models|Runtime Models]]**: Internal TypeScript interfaces (`TradeEvent`).
* **Related Stream Consumers & Sister Services**:
  * **[[entities/order-book-consumer|OrderBookConsumer]]**: Parallel consumer handling `nte.orderbook.snapshots`.
  * **[[entities/compliance-surveillance-monitor|Compliance Surveillance Monitor]]**: Sister service consuming corresponding market telemetry.
  * **[[entities/trade-settlement-system|Trade Settlement System]]**: Sister service handling post-trade clearing and settlement.

---

## Implementation Details

### Configuration

The consumer is instantiated with a dedicated Kafka client and consumer group:

```typescript
private kafka = new Kafka({ 
    clientId: 'mdg-trade-consumer', 
    brokers: [process.env.KAFKA_BROKERS || 'localhost:9092'] 
});
private consumer = this.kafka.consumer({ groupId: 'mdg-trade-group' });
```

### Ingestion Lifecycle

```
[Order Matching Engine]
          │
          ▼ (Kafka: nte.trades.matched)
+────────────────────────────────────────+
|             TradeConsumer              |
|  - Client: mdg-trade-consumer          |
|  - Group:  mdg-trade-group             |
+────────────────────────────────────────+
          │
          │ passes raw trade string
          ▼
+────────────────────────────────────────+
|             ClientManager              |
|  - Packages into TRADE_TICK envelope   |
|  - Iterates over active WS clients     |
+────────────────────────────────────────+
          │
          ▼
   [WebSocket Clients]
```

1. **Bootstrap**: Initialized in `src/index.ts` alongside [[entities/order-book-consumer|OrderBookConsumer]] and started via `await tradeConsumer.start()`.
2. **Subscription**: Connects to the Kafka broker and subscribes to `nte.trades.matched` without offset rewinds (`fromBeginning: false`).
3. **Execution Loop**: Uses `kafkajs` `consumer.run({ eachMessage })` to asynchronously process each message batch.
4. **Dispatch**: For each message containing a valid value, `clientManager.broadcastTrade(tradeData)` is invoked to fan out the tick.

---

## Wire Format & Runtime Types

Trades published to `nte.trades.matched` map directly to the `TradeTick` Protobuf definition in [[entities/protobuf-contracts|market_data.proto]]:

```protobuf
message TradeTick {
    string trade_id = 1;
    string symbol = 2;
    double price = 3;
    double quantity = 4;
    int64 matched_at_ns = 5;
    string taker_side = 6;
}
```

At runtime, this corresponds to the TypeScript `TradeEvent` model defined in [[entities/runtime-models|src/models/types.ts]], which is enveloped into `{ type: "TRADE_TICK", data: ... }` before transmission over [[concepts/websocket-broadcasting|WebSockets]] following the [[decisions/low-latency-egress-design|Low-Latency Egress Design]]. Detailed lifecycle flow is documented in [[concepts/data-lifecycle|Data Lifecycle]].