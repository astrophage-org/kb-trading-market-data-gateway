---
mission: SOL-3
title: 'Market data tracking visibility on the UI'
role: business
status: approved
version: 1
author: Akash Bajpai
ai_drafted: false
approved_at: 2026-09-24T12:31:22Z
---

# Business requirement: Market data tracking visibility on the UI

## The request
"Customer should be able to see how does the the tracking work on the UI"

## Problem
Customers and trading participants connected to the Nexus Trading Exchange (NTE) platform receive live order book depth updates and trade execution ticks via the Market Data Gateway (MDG) ([[kb:market-data-gateway/summaries/market-data-gateway-overview]]). However, the user interface currently provides no clear explanation or visual indication of how data tracking and market event propagation function across the system. 

When users look at the UI, they cannot see how order matching events move through distributed streaming topics (`nte.orderbook.snapshots` and `nte.trades.matched`) via the [[kb:market-data-gateway/entities/kafka-broker]] and fan out through the [[kb:market-data-gateway/summaries/market-data-gateway-overview]] WebSocket connections to their screen. Without visible tracking transparency, customers cannot easily confirm whether their session is actively tracking live market data or understand the stages of the [[kb:market-data-gateway/concepts/data-lifecycle]].

## Who is affected
- **External market participants and trading clients**: Customers relying on the UI to follow order book changes and trade execution feeds in real time.
- **Support and operations teams**: Client service teams responding to customer inquiries regarding how market data updates and trade ticks are tracked and delivered.

## What should change
- Introduce clear, accessible UI elements and explanatory views that illustrate how tracking works across the market data lifecycle.
- Expose clear visual status indicators on the client interface showing the active state of data streams (such as order book snapshots and trade tick updates).
- Provide descriptive tracking information on the UI detailing how events travel from matching engine generation, through broker message topics, to client WebSocket delivery.

## What "done" looks like
A customer navigating the UI can easily view and understand the end-to-end tracking mechanism for market data. The interface clearly displays how their live data stream is tracked and delivered from the platform to their screen, giving immediate confidence in data flow and operational status.

## Examples
- **Example 1: Reviewing tracking workflow**: A customer opens the market data view on the UI and clicks an information/tracking panel, which presents a plain-language summary showing how market depth and trade ticks are tracked and delivered in real time.
- **Example 2: Live tracking status**: A trader monitors their active session on the UI and sees visual indicators confirming that both the order book snapshot feed and trade match stream are actively tracking and receiving live updates from the gateway.

## Verification checklist
- [ ] The UI displays a clear explanation of how market data tracking operates from ingestion to client delivery.
- [ ] Customers can view real-time tracking and connection status for market data streams directly on the interface.
- [ ] Support documentation or help references within the UI guide users through the tracking lifecycle.
- [ ] Customer should be able to see how does the the tracking work on the UI.
