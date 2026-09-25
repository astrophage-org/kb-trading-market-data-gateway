---
mission: SOL-4
title: 'Daily Customer Sales Metrics on Dashboard'
role: product
status: ai_drafted
version: 1
author: Sol
ai_drafted: true
---

# Product spec: Daily Customer Sales Dashboard Metrics

## Goal
Provide a dedicated dashboard view and metrics aggregation service that displays daily sales metrics—including total trade count, executed volume, and gross notional sales value—per customer for current and historical dates, eliminating manual reporting and direct log querying.

## User stories
- **US-1**: As a Sales or Account Manager, I want to view a daily summary of sales and executed trading metrics broken down by customer, so that I can evaluate client engagement and daily trading volume.
- **US-2**: As an Operations Analyst, I want to filter and search customer sales metrics by specific dates or date ranges, so that I can audit historical daily activity without manual database or log extraction.
- **US-3**: As an Exchange Operations User, I want daily customer metrics derived accurately from completed trade executions (`nte.trades.matched` [[kb:market-data-gateway/decisions/independent-consumer-groups]]), so that the dashboard reflects verified matched trade activity.

## Acceptance criteria
- **AC-1: Daily Sales View on Dashboard**
  - **Given** an authenticated user with dashboard access,
  - **When** they navigate to the customer sales dashboard section,
  - **Then** the interface displays a tabular or summary view of customers with their aggregated daily metrics for the selected date (defaulting to the current trading date).
- **AC-2: Metric Aggregations per Customer**
  - **Given** executed trade events processed from the trade stream (`nte.trades.matched` [[kb:market-data-gateway/decisions/independent-consumer-groups]]),
  - **When** customer sales metrics are aggregated for a specific date,
  - **Then** each customer record displays:
    1. Customer / Account Identifier
    2. Date (YYYY-MM-DD)
    3. Total Notional Value (sum of `execution price * quantity`)
    4. Total Executed Volume (sum of filled contract/unit quantities)
    5. Completed Trade Count (total matched trade records)
- **AC-3: Date Picker & Historical Selection**
  - **Given** a user viewing the daily sales dashboard,
  - **When** the user selects a past date using the date picker,
  - **Then** the dashboard updates to display finalized historical customer sales aggregations for that specific date.
- **AC-4: Search and Column Sorting**
  - **Given** the daily sales view with multiple customer records,
  - **When** the user searches by customer identifier or sorts by notional value / trade volume,
  - **Then** the list filters and orders matching records immediately without losing the selected date filter.

## Edge cases
- **Zero-Trade Customers on Selected Day**: A customer with an active account has zero matched executions on the selected date. The system should either display zeroed metrics (`$0.00` value, `0` volume, `0` trades) or omit them when filtered by "Active Trades Only".
- **Trade Stream Bursts & Ingestion Lag *(from the map)***: High-frequency trade volume spikes on `nte.trades.matched` ingested by the trade consumer group (`mdg-trade-group` [[kb:market-data-gateway/decisions/independent-consumer-groups]]) may cause transient aggregation lag for intraday live views. The dashboard should indicate last-updated timestamp.
- **Partition Rebalance Delays *(from the map)***: Temporary consumer partition rebalances on the Kafka broker cluster [[kb:market-data-gateway/concepts/ingestion-egress-decoupling]] should not corrupt daily aggregation state; pending trades must be processed sequentially without duplicate counting.
- **Multi-Symbol Orders per Customer**: A customer executing trades across multiple symbols within the same trading day must have all transactions correctly summed into their daily aggregated total notional value and total volume.

## Out of scope
- Real-time Level 2 order book depth rendering or depth snapshot hydration via `nte.orderbook.snapshots` [[kb:market-data-gateway/decisions/redis-l2-state-caching]].
- Automated invoicing, tax calculation, or billing adjustments.
- Direct manual editing, trade cancellation, or modification of completed trade records from the dashboard.

## Success metric
- 100% reduction in manual ad-hoc data requests for daily customer sales numbers.
- Dashboard daily sales query and load time under 1.5 seconds for any selected calendar date.

## Priority
**P2** — High operational and business value. While core trade execution and order book distribution proceed independently via the [[kb:market-data-gateway/concepts/ingestion-egress-decoupling|Market Data Gateway]], daily sales reporting is essential for commercial account management and operational reporting.

## Verification checklist
- [ ] AC-1: Dashboard displays daily customer sales section on current trading day.
- [ ] AC-2: Metrics display Customer ID, Date, Total Notional Value, Total Volume, and Trade Count derived from matched trades.
- [ ] AC-3: Date picker allows selecting and displaying historical daily aggregations.
- [ ] AC-4: Search by customer identifier and sorting by metric columns work as expected.
- [ ] Edge Case: Customers with zero trades on a given day are handled gracefully without application error.
- [ ] Edge Case: Trade stream spikes on `nte.trades.matched` do not crash aggregation and display updated timestamp *(from the map)*.
- [ ] Edge Case: Kafka partition rebalances do not create duplicate trade metrics *(from the map)*.
- [ ] Edge Case: Multi-symbol transactions for a single customer sum correctly.
- [ ] Success Metric: Manual data extraction requests reduced to 0 and page query load latency remains under 1.5 seconds.
