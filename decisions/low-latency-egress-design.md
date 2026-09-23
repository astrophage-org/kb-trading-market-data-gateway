# Low-Latency Egress Design: In-Memory Fan-Out and Backpressure Mitigation

## Status

**Accepted**

---

## Context

The **Market Data Gateway (MDG)** is responsible for distributing real-time Level 2 order book updates (`L2_UPDATE`) and execution ticks (`TRADE_TICK`) from the [[entities/order-matching-engine]] to external market participants, institutional trading desks, and UI subscribers.

In financial exchange architectures, market data dissemination requires:
1. **Low, Deterministic Latency**: Minimizing processing jitter between Kafka event consumption and WebSocket packet transmission.
2. **Market Data Fairness**: Ensuring updates are dispatched to all active subscribers simultaneously without slow connections stalling message flow to faster peers.
3. **Ingestion Isolation**: Preventing downstream network latency, slow client consumers, or socket disconnections from creating backpressure that blocks [[entities/order-book-consumer]] or [[entities/trade-consumer]] Kafka poll loops.

Initial designs considered per-client message queues with dedicated worker threads, intermediate Pub/Sub routing layers, and direct in-memory iteration. Given Node.js single-threaded event loop characteristics and high-frequency market data loads, an optimal broadcast architecture was required to maximize throughput while preventing socket backpressure degradation.

---

## Decision

We decided to implement an **in-memory non-blocking fan-out broadcast engine** encapsulated within the [[entities/client-manager]], combined with proactive connection state validation and strict decoupling from the ingestion tier.

### 1. In-Memory Connection Pooling
The [[entities/client-manager]] maintains active client connections in an in-memory `Set<WebSocket>` data structure. 
* Connection registration is handled instantaneously upon the `connection` event.
* Connections are automatically unregistered via socket lifecycle listeners (`ws.on('close', ...)`).

```typescript
// Connection management inside ClientManager
this.wss.on('connection', (ws) => {
    this.clients.add(ws);
    ws.on('close', () => this.clients.delete(ws));
});
```

### 2. Fast-Path Fan-Out Loop
Egress broadcasts (`broadcastOrderBook` and `broadcastTrade`) iterate synchronously over the active socket set. To eliminate conversion and transformation overhead in the egress hot-path, serialized JSON envelopes (`[[entities/runtime-models]]`) are prepared and dispatched directly across open sockets.

```typescript
broadcastOrderBook(snapshot: string) {
    for (const client of this.clients) {
        if (client.readyState === WebSocket.OPEN) {
            client.send(JSON.stringify({ type: 'L2_UPDATE', data: snapshot }));
        }
    }
}
```

### 3. Backpressure Mitigation & State Filtering
To prevent degraded or half-closed connections from blocking the broadcast pipeline or causing memory leaks:
* **State Check**: Sockets are validated for `WebSocket.OPEN` status prior to every write attempt. Non-open sockets (e.g., `CONNECTING`, `CLOSING`, `CLOSED`) are bypassed.
* **Non-Blocking Ingestion**: Consumer loops in [[entities/order-book-consumer]] and [[entities/trade-consumer]] invoke the broadcast methods asynchronously without awaiting individual client TCP acknowledgments, enforcing [[concepts/ingestion-egress-decoupling]].
* **Lifecycle Cleanup**: Broken or terminated connections trigger immediate eviction from the client pool via close handlers.

---

## Consequences & Trade-Offs

### Positive
* **Sub-Millisecond Egress Dispatch**: In-memory iteration over native `ws` sockets introduces near-zero latency overhead between Kafka receipt and wire transmission.
* **Decoupled Failure Domains**: A stalled or disconnecting external client cannot stall Kafka partition consumption or affect message delivery to other connected market participants.
* **Simplicity & Predictability**: Eliminating complex intermediate buffering layers reduces garbage collection pressure and minimizes heap memory fragmentation under high tick rates.

### Negative & Mitigations
* **Kernel TCP Buffer Saturation**: If a connected client experiences severe downstream network congestion, data may accumulate in the Node.js socket write buffer (`ws.bufferedAmount`).
  * *Mitigation*: Planned enhancements include monitoring `ws.bufferedAmount` thresholds to aggressively terminate severely lagging clients in compliance with exchange fairness rules.
* **Process Scaling Limits**: Because connections are pooled in-process memory, horizontal scaling across multiple MDG instances requires a layer-4 load balancer (such as NGINX or AWS NLB) with WebSocket stickiness, while Kafka consumer groups handle parallel stream consumption (see [[decisions/independent-consumer-groups]]). Initial state hydration for newly connected clients is offloaded to [[entities/redis-cache]] (see [[decisions/redis-l2-state-caching]]).

---

## Related Documents

* [[concepts/websocket-broadcasting]] — Implementation patterns for WebSocket lifecycle management and fan-out distribution.
* [[concepts/ingestion-egress-decoupling]] — Architectural boundary between Kafka consumption and client egress.
* [[concepts/data-lifecycle]] — Full lifecycle from matching engine execution to external delivery.
* [[entities/client-manager]] — Component implementing the in-memory broadcast engine.
* [[decisions/independent-consumer-groups]] — Consumer group isolation strategy for incoming streams.
* [[decisions/redis-l2-state-caching]] — State hydration architecture for incoming WebSocket connections.
* [[summaries/market-data-gateway-overview]] — High-level MDG architectural and topological overview.