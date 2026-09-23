<!-- anchor: docs/architecture.md:L1-L100 sha:HEAD -->

# Compliance Surveillance Monitor

The **Compliance Surveillance Monitor** (`@astrophage/compliance-surveillance-monitor`) is a mission-critical sister service within the **Global Financial Markets Group (GFMG) Nexus Trading Exchange (NTE)** ecosystem. It acts as an independent downstream monitoring and regulatory audit engine that inspects market event streams, execution integrity, dissemination fairness, and participant trading behavior in real time.

Operating downstream from both the [[entities/order-matching-engine]] and the [[summaries/market-data-gateway-overview]], the surveillance engine verifies that public dissemination via [[concepts/websocket-broadcasting]] adheres to exchange fairness mandates and regulatory rules against market manipulation.

---

## Responsibilities

The primary responsibilities of the Compliance Surveillance Monitor include:

* **Market Abuse & Anomaly Detection**: Ingesting trade ticks and Level 2 book updates to detect illicit trading patterns, such as spoofing, layering, wash trading, quote stuffing, and front-running.
* **Fair Dissemination & Latency Arbitrage Auditing**: Monitoring telemetry feeds (e.g., `nte.mdg.telemetry`) from the [[summaries/market-data-gateway-overview]] to verify that the [[entities/client-manager]] broadcasts market data fairly across all connected WebSocket subscribers without preferential latency leaks.
* **Execution Timestamp Reconciliation**: Cross-referencing matching engine execution timestamps (`matched_at_ns` from `nte.trades.matched`) with MDG egress dispatch timestamps to measure edge transport latency and identify network backpressure anomalies.
* **Regulatory Record Keeping & Audit Trail Generation**: Constructing a deterministic, tamper-evident audit trail of all order book state changes (`nte.orderbook.snapshots`) and trade executions (`nte.trades.matched`) for financial regulatory authorities.
* **Circuit Breaker & Market Integrity Alerts**: Generating real-time compliance alerts or triggering automated throttle interventions when abnormal volatility, order book depletion, or market anomalies occur.

---

## Dependencies

The Compliance Surveillance Monitor relies on the following components and platform interfaces:

* **[[entities/order-matching-engine]]**: Upstream generator of canonical trade match events (`nte.trades.matched`) and aggregated L2 order book snapshots (`nte.orderbook.snapshots`).
* **[[summaries/market-data-gateway-overview]]**: Edge gateway emitting broadcast telemetry (`nte.mdg.telemetry`), connection state changes, and socket distribution metrics.
* **[[entities/kafka-broker]]**: Distributed messaging backbone providing partitioned, ordered event streams across shared consumer topics.
* **[[entities/protobuf-contracts]]**: Schema definitions (`protos/market_data.proto`) defining structured serialization for `OrderBookSnapshot`, `Level`, and `TradeTick` messages.
* **[[entities/trade-settlement-system]]**: Downstream clearing and settlement counterparty for post-trade clearing validation and participant position reconciliations.

---

## Integration Architecture & Data Flow

The Compliance Surveillance Monitor consumes data from multiple points in the [[concepts/data-lifecycle]] to perform dual-entry surveillance and egress fairness verification:

```
                          +-------------------------------+
                          |   `Order Matching Engine`   |
                          +-------------------------------+
                                     |         |
         (nte.orderbook.snapshots)   |         |   (nte.trades.matched)
       +-----------------------------+         +-----------------------------+
       |                                                                     |
       v                                                                     v
+-------------------------------+                         +--------------------------------------+
|  `Market Data Gateway (MDG)` |                         |    Compliance Surveillance Monitor   |
|   - `OrderBookConsumer`     |                         |   - Real-Time Trade Surveillance     |
|   - `TradeConsumer`         |                         |   - Order Book Spoofing Detection    |
|   - `ClientManager`         |                         |   - Latency / Fairness Audit Engine  |
+-------------------------------+                         +--------------------------------------+
       |                                                                     ^
       | (nte.mdg.telemetry)                                                 |
       +---------------------------------------------------------------------+
```

### 1. Ingestion Topics
* **`nte.trades.matched`**: Ingested asynchronously to evaluate execution prices, volume thresholds, participant crossing, and transaction patterns.
* **`nte.orderbook.snapshots`**: Consumed to maintain an independent shadow order book, verifying depth consistency and monitoring quote cancellation rates.
* **`nte.mdg.telemetry`**: Emitted by MDG runtime instances to track client pool metrics, socket write latencies, and dropped connection statistics from the [[entities/client-manager]].

### 2. Protobuf Wire Compatibility
Surveillance services consume the identical [[entities/protobuf-contracts]] defined in `protos/market_data.proto`:
* **`nte.marketdata.TradeTick`**:
  * `trade_id` (`string`): Unique trade execution identifier.
  * `symbol` (`string`): Ticker instrument.
  * `price` (`double`): Matched execution price.
  * `quantity` (`double`): Filled quantity.
  * `matched_at_ns` (`int64`): Nanosecond-precision matching timestamp.
  * `taker_side` (`string`): Aggressor side (`BUY` / `SELL`).
* **`nte.marketdata.OrderBookSnapshot`**:
  * `symbol` (`string`): Ticker instrument.
  * `bids` / `asks` (`repeated Level`): Price, quantity, and order count at each depth tier.
  * `timestamp_ns` (`int64`): Nanosecond snapshot creation timestamp.

---

## Relationship with MDG

While MDG transforms internal Kafka events into public WebSocket envelopes (`L2_UPDATE` and `TRADE_TICK` as defined in [[entities/runtime-models]]), the Compliance Surveillance Monitor acts as the supervisory observer. 

Because MDG implements [[concepts/ingestion-egress-decoupling]] and distinct consumer groups via [[decisions/independent-consumer-groups]], the surveillance cluster operates completely independently from MDG's client egress performance, preventing surveillance analytical workloads from impacting public market data latency.