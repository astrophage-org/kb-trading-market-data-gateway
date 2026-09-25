---
mission: SOL-4
title: 'Daily Customer Sales Metrics on Dashboard'
role: business
status: draft
version: 1
author: Akash Bajpai
ai_drafted: false
---

# Business requirement: Daily Customer Sales Metrics on Dashboard

## The request
> "I want the data related to customer sales per day on the dasboard."

## Problem
Currently, business managers, account executives, and operations teams cannot view aggregated daily sales data per customer directly on their dashboard. While the underlying exchange and gateway process trade executions (such as matched trade streams from `nte.trades.matched` [[kb:market-data-gateway/decisions/independent-consumer-groups]]), there is no consolidated visual summary showing how much each customer has transacted or purchased on a day-by-day basis. Stakeholders must either request manual reports or cross-reference raw logs to determine daily customer-level sales volumes.

## Who is affected
- **Sales and Account Managers**: Need quick visibility into daily customer purchasing volume and account activity to manage client relationships.
- **Operations & Business Analysts**: Need reliable daily sales aggregations on the dashboard without having to extract raw trade records manually.
- **Executive Leadership**: Needs high-level visibility into daily sales trends across the customer base.

## What should change
- Introduce a dedicated daily customer sales view or widget on the dashboard.
- Display total sales metrics (such as trade count, total volume, and notional sales value) aggregated per customer for each day.
- Allow users to select specific dates or date ranges to compare daily customer sales over time.
- Provide sorting and search capabilities so users can quickly find a specific customer's daily totals.
- *Open Questions*:
  - What exact dashboard interface/application will host this widget?
  - Should sales data update in near real-time throughout the trading day, or only finalize at end-of-day settlement?
  - Which specific metrics constitute "sales" for each customer (e.g., executed buy/sell volume, gross notional value, or net revenue/fees)?

## What "done" looks like
A business user can log into the dashboard, navigate to the sales section, and immediately view a clear breakdown of daily sales figures for each customer. The user can switch dates and verify accurate daily totals without manual calculations or external reporting tools.

## Examples
- **Example 1 (Current Day View)**: A sales manager opens the dashboard on May 10th and sees a table listing all active customers, showing Customer A with 120 sales transactions totaling $450,000 and Customer B with 45 transactions totaling $120,000 for the day.
- **Example 2 (Historical Day View)**: An analyst selects May 8th from the date picker to review historical sales; the dashboard displays the finalized sales totals for each customer on that specific date.

## Verification checklist
- [ ] I can log into the dashboard and locate a view displaying daily sales broken down by customer.
- [ ] I can change the selected date to review customer sales figures for previous days.
- [ ] The aggregated daily sales totals match the actual completed customer transactions for the chosen day.
- [ ] The data related to customer sales per day is now clearly visible and accessible on the dashboard.
