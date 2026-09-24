---
mission: SOL-3
title: 'Market data tracking visibility on the UI'
role: product
status: ai_drafted
version: 1
author: Sol
ai_drafted: true
---

# Product spec: Market data tracking visibility on the UI

## Goal
Provide market participants, traders, and support teams with comprehensive on-screen visibility into how market data tracking and event distribution operate across the Nexus Trading Exchange (NTE) platform. The interface will visually explain and display the real-time [[kb:market-data-gateway/concepts/data-lifecycle]], tracing events from the matching core and [[kb:market-data-gateway/entities/kafka-broker]] streaming topics down to edge WebSocket delivery via the [[kb:market-data-gateway/summaries/market-data-gateway-overview]].

## User stories
- **As a market participant or trader**, I want to view an intuitive explanation and visual representation of how market depth and trade ticks are tracked across the platform, so that I have full transparency and confidence in the integrity and freshness of the data on my screen.
- **As an active UI user**, I want to see real-time tracking indicators for active order book snapshot and trade execution streams, so that I can immediately verify whether my session is actively tracking live market events.
- **As a client support analyst**, I want the UI to expose detailed tracking stage breakdowns (from message topics to client egress), so that I can quickly guide customers and diagnose stream delivery questions without needing backend access.

## Acceptance criteria
- **AC-1**: Given a user navigating the market data trading UI, When they access the tracking info view or tracking panel, Then the UI displays an end-to-end explanation of the [[kb:market-data-gateway/concepts/data-lifecycle]], detailing how events flow from generation in the matching engine through the [[kb:market-data-gateway/entities/kafka-broker]] to the [[kb:market-data-gateway/summaries/market-data-gateway-overview]] client connection pool.
- **AC-2**: Given an active market data session, When real-time order book snapshots and trade executions are received over WebSocket, Then the UI displays live tracking status indicators confirming active tracking on both the `nte.orderbook.snapshots` and `nte.trades.matched` event pipelines.
- **AC-3**: Given a user inspecting specific data streams in the tracking panel, When they select a stream (e.g., Level 2 book updates or trade execution ticks), Then the UI presents technical tracking details including Kafka topic names (`nte.orderbook.snapshots`, `nte.trades.matched`), consumer group isolation context (`mdg-orderbook-group`, `mdg-trade-group`), and egress message types (`L2_UPDATE`, `TRADE_TICK`).
- **AC-4**: Given a user viewing the tracking panel, When they hover over or expand any lifecycle stage (Ingestion, Isolation, Egress), Then contextual tooltips explain the role of each stage in maintaining deterministic distribution and decoupling internal brokers from client connections.

## Edge cases
- **WebSocket connection disruption**: When the client WebSocket connection to the [[kb:market-data-gateway/summaries/market-data-gateway-overview]] drops or degrades, the UI tracking indicators must immediately transition to a disconnected or degraded state rather than showing active tracking.
- **Upstream consumer group lag *(from the map)***: When broker message consumption by `mdg-orderbook-group` or `mdg-trade-group` on [[kb:market-data-gateway/entities/kafka-broker]] topics experiences processing latency, the tracking view must indicate that live stream delivery is delayed.
- **Low-volume instrument dormancy**: When a trading pair has continuous Level 2 snapshots on `nte.orderbook.snapshots` but zero trade executions on `nte.trades.matched`, the tracking UI must clearly distinguish between an idle trade stream and an inactive or disconnected tracking pipeline.

## Out of scope
- Modifying backend Kafka partition layouts, broker configurations, or matching engine logic.
- Granting users the ability to alter consumer group offsets or disconnect other client sessions.
- Ingesting and visualizing historical message replay or compliance logs outside live market tracking.

## Success metric
- 100% of UI users are able to access the tracking visibility panel and view live stream status.
- A 30% reduction in customer support tickets regarding market data stream confusion and tracking status within 30 days of deployment.

## Priority (P1–P4) and why
**P2**: While core trading and live data streaming functions operate without this feature, tracking visibility directly impacts customer trust, market fairness transparency, and support operational efficiency on the NTE platform.

## Verification checklist
- [ ] AC-1: Data tracking panel displays the end-to-end data lifecycle from matching engine through broker topics to client WebSocket egress.
- [ ] AC-2: Live tracking indicators accurately reflect active ingestion and delivery for both `nte.orderbook.snapshots` and `nte.trades.matched`.
- [ ] AC-3: Detailed stream inspector displays topic names, consumer groups, and egress message formats (`L2_UPDATE` and `TRADE_TICK`).
- [ ] AC-4: Contextual lifecycle stage tooltips provide clear explanations of broker decoupling and egress fan-out.
- [ ] Edge case: Tracking status indicators immediately update to disconnected/degraded during WebSocket connection loss.
- [ ] Edge case: Tracking view indicates latency when consumer group lag occurs on Kafka topics *(from the map)*.
- [ ] Edge case: Low-volume pairs with no matched trades display an idle stream status rather than a broken tracking connection.
- [ ] Success metric: 100% UI tracking visibility availability achieved with measurable reduction in stream-related support inquiries.
