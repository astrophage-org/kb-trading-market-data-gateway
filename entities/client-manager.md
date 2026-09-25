<!-- anchor: src/websockets/ClientManager.ts:L1-L100 sha:HEAD -->

# ClientManager

The `ClientManager` is the core WebSocket egress engine within the **Market Data Gateway (MDG)** (`@astrophage/market-data-gateway`). Located in `src/websockets/ClientManager.ts`, it encapsulates WebSocket server lifecycle management, in-memory connection pooling, socket state monitoring, and the fan-out broadcasting of real-time market data to connected external participants and trading interfaces.

As detailed in [[concepts/ingestion-egress-decoupling]], `ClientManager` abstracts the downstream client distribution layer from internal messaging protocols, serving as the egress target for [[entities/order-book-consumer]] and [[entities/trade-consumer]].

---

## Responsibilities

* **WebSocket Server Lifecycle Management**: Instantiates and maintains the underlying `ws.WebSocketServer` instance bound to the configured network port (e.g., TCP port `8080` during runtime bootstrap in [[index]]).
* **Connection Pool Management**: Tracks active client connections within an in-memory `Set<WebSocket>`, automatically enrolling new clients on `'connection'` events and pruning disconnected sockets on `'close'` events.
* **Socket State Validation**: Enforces safety checks on socket readiness (`client.readyState === WebSocket.OPEN`) before dispatching payloads, preventing runtime exceptions and connection stalls against dead or closing sockets.
* **Market Data Fan-out Dispatch**: Formats and broadcasts outbound payloads across all active sockets:
  * **Order Book Snapshots**: Dispatches Level 2 depth envelopes (`L2_UPDATE`) via `broadcastOrderBook()`.
  * **Trade Execution Ticks**: Dispatches trade match envelopes (`TRADE_TICK`) via `broadcastTrade()`.
* **Latency & Fairness Preservation**: Executes non-blocking iterative dispatch loops to deliver market ticks uniformly across all subscribed clients, upholding the system's low-latency dissemination requirements (see [[decisions/low-latency-egress-design]]).

---

## Dependencies

* **`ws` (`WebSocketServer`, `WebSocket`)**: High-performance Node.js WebSocket library utilized for transport connection handling and streaming writes.
* **[[entities/order-book-consumer]]**: Invokes `ClientManager.broadcastOrderBook()` upon ingesting aggregated L2 snapshots from the `nte.orderbook.snapshots` Kafka topic.
* **[[entities/trade-consumer]]**: Invokes `ClientManager.broadcastTrade()` upon ingesting executed trade ticks from the `nte.trades.matched` Kafka topic.
* **[[entities/runtime-models]]**: Defines the logical structure and envelope schemas (`L2_UPDATE`, `TRADE_TICK`) used during serialization.
* **[[entities/redis-cache]] / [[concepts/state-hydration]]**: Complementary infrastructure utilized during initial handshake procedures to hydrate new clients with the latest order book baseline.

---

## Architecture & Implementation

```
   `OrderBookConsumer`                 `TradeConsumer`
            │                                    │
 broadcastOrderBook(data)             broadcastTrade(data)
            │                                    │
            ▼                                    ▼
  +─────────────────────────────────────────────────────────+
  |                      ClientManager                      |
  |                                                         |
  |  - wss: WebSocketServer (Port 8080)                     |
  |  - clients: Set<WebSocket>                              |
  +─────────────────────────────────────────────────────────+
            │                                    │
  [JSON: L2_UPDATE]                    [JSON: TRADE_TICK]
            │                                    │
            +─────────────────┬──────────────────+
                              │
                    (Fan-out Broadcast)
                              │
     ┌────────────────────────┼────────────────────────┐
     ▼                        ▼                        ▼
[Client Socket 1]        [Client Socket 2]        [Client Socket N]
(WebSocket.OPEN)         (WebSocket.OPEN)         (WebSocket.OPEN)
```

### 1. Connection Pool Lifecycle
The `ClientManager` maintains an in-memory registry of active clients using a standard ES6 `Set<WebSocket>` collection:

```typescript
export class ClientManager {
    private wss: WebSocketServer;
    private clients: Set<WebSocket> = new Set();

    constructor(port: number) {
        this.wss = new WebSocketServer({ port });
        this.wss.on('connection', (ws) => {
            this.clients.add(ws);
            ws.on('close', () => this.clients.delete(ws));
        });
    }
    // ...
}
```

* When a client connects via WebSocket handshake, it is registered in `this.clients`.
* The `'close'` event listener ensures that disconnected sockets are immediately evicted from memory, preventing memory leaks and avoiding unnecessary iterations during broadcast cycles.

### 2. Message Dispatch Envelopes

The component provides two dedicated broadcast methods that wrap raw stringified payloads into standard JSON envelopes:

```typescript
broadcastOrderBook(snapshot: string) {
    for (const client of this.clients) {
        if (client.readyState === WebSocket.OPEN) {
            client.send(JSON.stringify({ type: 'L2_UPDATE', data: snapshot }));
        }
    }
}

broadcastTrade(trade: string) {
    for (const client of this.clients) {
        if (client.readyState === WebSocket.OPEN) {
            client.send(JSON.stringify({ type: 'TRADE_TICK', data: trade }));
        }
    }
}
```

#### Outbound Wire Format

| Event Type | Envelope Key | Payload Schema Origin | Description |
| :--- | :--- | :--- | :--- |
| `L2_UPDATE` | `type: "L2_UPDATE"` | `nte.marketdata.OrderBookSnapshot` / `OrderBookEvent` | Aggregated bid and ask price levels produced by [[entities/order-matching-engine]]. |
| `TRADE_TICK` | `type: "TRADE_TICK"` | `nte.marketdata.TradeTick` / `TradeEvent` | Executed trade match data including execution price, quantity, timestamp, and taker side. |

---

## Operational Characteristics

* **Egress Decoupling**: By placing the fan-out loop directly inside `ClientManager`, upstream Kafka consumer threads in [[entities/order-book-consumer]] and [[entities/trade-consumer]] are not directly coupled to individual network sockets.
* **State Hydration Integration**: While streaming updates are broadcast in real-time, new connections can establish initial state by fetching historical snapshots from [[entities/redis-cache]] as outlined in [[decisions/redis-l2-state-caching]].
* **Downstream Ecosystem**: Serves as the primary public egress gateway for external market participants, while internal audit feeds are handled separately by [[entities/compliance-surveillance-monitor]] and [[entities/trade-settlement-system]].

---

## Related Concepts & Components

* **Data Pipeline**: [[concepts/data-lifecycle]]
* **Broadcasting Logic**: [[concepts/websocket-broadcasting]]
* **Architectural Overview**: [[summaries/market-data-gateway-overview]]
* **Infrastructure Layer**: [[entities/kafka-broker]], [[entities/redis-cache]]
* **Protobuf Wire Contracts**: [[entities/protobuf-contracts]]