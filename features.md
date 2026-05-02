# Subscription Analytics Platform — Feature & Functionality Survey

> Candidate #391 · Researched: 2026-05-02

## Solutions Analysed

| Tool | Type | Licence / Model | URL |
|------|------|-----------------|-----|
| ChartMogul | SaaS | Commercial (free tier to 100 customers) | https://chartmogul.com |
| Baremetrics | SaaS | Commercial | https://baremetrics.com |
| ProfitWell Metrics (Paddle) | SaaS | Free (Paddle-acquired) | https://www.profitwell.com |
| Stripe Billing | Managed billing + analytics | Commercial (% of revenue) | https://stripe.com/billing |
| Chargebee | SaaS billing + analytics | Commercial | https://www.chargebee.com |

## Feature Analysis by Solution

### ChartMogul

**Core features**
- MRR ledger constructed from billing events: new MRR, expansion, contraction, churn, and reactivation movements tracked daily
- Cohort Net MRR Retention: tracks how much recurring revenue is retained in subsequent periods after a cohort's subscription starts, netting expansion and reactivation against contraction and churn
- Segmentation of MRR movements by plan, geography, customer attribute, or custom tag
- Benchmarking dashboard comparing key metrics against a pool of 30,000+ SaaS companies anonymised by revenue band and vertical
- Customer Journey view: full subscription history per customer including plan changes, pauses, and cancellations

**Differentiating features**
- Benchmark database is uniquely large, offering statistically meaningful peer comparisons for churn rate, LTV, and ARPU
- Data platform integrations: Salesforce, HubSpot, and Segment for enriching billing records with CRM and product-usage attributes
- Custom attributes and tags on customer records enable bespoke segmentation unavailable in generic BI tools

**UX patterns**
- MRR waterfall chart as the primary dashboard (colour-coded by movement type)
- Cohort retention grid with heatmap colouring for quick identification of degrading cohorts
- Customer list with drill-down to per-customer subscription timeline

**Integration points**
- Native connectors for Stripe, Chargebee, Recurly, Paddle, Braintree, and others
- REST API for custom data import from proprietary billing systems
- Segment and Rudderstack as customer data sources

**Known gaps**
- Voluntary vs. involuntary churn attribution requires manual tagging in some cases; automated dunning-stage classification is limited
- No built-in product-usage data — expansion signal detection requires integration with a separate analytics tool
- AI-driven anomaly alerting is basic; real-time MRR movement alerts require configuration of webhook-based monitoring

**Licence / IP notes**
- Proprietary SaaS. No open-source components. Data portability via API and CSV export.

---

### Baremetrics

**Core features**
- MRR segmentation and cohort analysis across plan types and subscription tiers
- Revenue-retention cohort view showing which signup months produce highest-LTV customers
- Churn breakdown by plan, showing that annual subscribers typically churn at lower rates than monthly
- Trial insights: conversion rate from trial to paid by cohort and acquisition channel
- Recover: automated dunning email sequences for failed payments targeting involuntary churn

**Differentiating features**
- Recover (dunning automation) built into the same product as analytics — rare combination
- Customer segmentation by LTV, plan, churn risk, and custom attributes with exportable lists
- People section: individual customer profiles with communication timeline

**UX patterns**
- Dashboard metrics tiles with configurable date range and comparison period
- Forecast module projecting MRR trajectory based on current growth and churn rates
- Slack integration for daily MRR digests and instant churn notifications

**Integration points**
- Stripe (primary), Braintree, Recurly, and Paddle connectors
- Slack and email alerting
- CSV export for custom analysis

**Known gaps**
- Stripe-centric architecture; non-Stripe billing systems have limited or no native support
- Benchmark data pool is smaller than ChartMogul's
- No data warehouse integration or SQL access to underlying data

**Licence / IP notes**
- Proprietary SaaS. Data export via CSV and API.

---

### ProfitWell Metrics (Paddle)

**Core features**
- Zero-cost subscription analytics covering MRR, churn, LTV, and revenue retention
- Bank-level data accuracy claim for Stripe-connected accounts via direct API access
- Subscription Correction feature flags and corrects data anomalies in billing event streams
- Retain: AI-driven churn intervention offering targeted discount and pause offers at the cancellation moment

**Differentiating features**
- Free at all tiers with no usage caps — positions as infrastructure-level service rather than premium analytics tool
- Retain (paid add-on) uses behavioural ML to determine optimal intervention for each churning customer
- Benchmark data from Paddle's own large SaaS customer base

**UX patterns**
- Metric cards with sparkline trends and period-over-period comparisons
- Cancellation flow insights showing which exit reasons and plan types correlate with churn
- Customer segmentation by subscription health score

**Integration points**
- Stripe and Paddle native integrations; limited connector library compared with ChartMogul
- Salesforce via Retain module

**Known gaps**
- Effectively a Stripe or Paddle-ecosystem product; other billing systems are not well supported
- Deep analytics (cohort grids, custom segmentation) require Retain paid tier
- Owned by Paddle since acquisition; product roadmap tied to Paddle's strategic priorities

**Licence / IP notes**
- Proprietary SaaS. Owned by Paddle; no open-source components.

---

## Cross-Cutting Feature Themes

### Table-Stakes Features
- MRR waterfall chart breaking down new, expansion, contraction, churn, and reactivation movements
- Monthly Recurring Revenue (MRR), Annual Recurring Revenue (ARR), Customer Lifetime Value (LTV), and Average Revenue Per User (ARPU) as core metrics
- Native connectors for Stripe, Chargebee, and Recurly as primary billing sources
- Cohort retention analysis showing net revenue retention by signup period
- Customer-level subscription timeline drill-down

### Differentiating Features
- Voluntary vs. involuntary churn classification with dunning-stage breakdown
- Benchmark comparison against anonymised peer companies in the same revenue band
- AI-driven churn intervention at the cancellation moment (ProfitWell Retain)
- Product-usage signal integration for expansion opportunity detection
- Real-time MRR movement alerts for anomalous activity

### Underserved Areas / Opportunities
- Dunning-stage analytics: tracking which payment-failure recovery steps (retry 1, retry 2, email, SMS) recover what share of involuntary churn
- Expansion signal detection: correlating product-usage milestones with upgrade probability to enable proactive outreach
- Multi-currency MRR normalisation with configurable base currency and historical FX rate preservation
- Embedded analytics for SaaS founders who want to display subscription metrics inside their own product dashboard

### AI-Augmentation Candidates
- Automated churn risk scoring combining billing patterns, product usage, and support interaction history
- Natural-language MRR commentary: "MRR declined 3.2% this month primarily due to annual-plan churn in the SMB segment"
- Forecast modelling under multiple growth and churn scenarios with confidence intervals
- Anomaly detection on daily MRR movements alerting teams to data pipeline issues before monthly reporting

## Legal & IP Summary

All mature tools in this category (ChartMogul, Baremetrics, ProfitWell) are proprietary SaaS platforms. The core analytical methods — MRR waterfall decomposition, cohort net-retention calculation, and LTV estimation — are standard accounting and actuarial techniques available in published literature with no patent encumbrances. Open-source tools referenced in the research (dbt, Metabase, Apache Superset) are available under Apache 2.0 or MIT licences and can form the analytical backbone of a new platform. Benchmark database construction would require a proprietary data-sharing agreement with participating companies; the ChartMogul and ProfitWell benchmark datasets themselves are trade secrets. A new entrant building on open-source BI components and standard subscription accounting methodology faces no legal barriers.

## Recommended Feature Scope

**Must-have (MVP)**:
- MRR ledger constructed from billing events with five-movement waterfall (new, expansion, contraction, churn, reactivation)
- Native connectors for Stripe, Chargebee, and Recurly with automated event normalisation
- Cohort net-MRR-retention grid by signup month and plan
- Customer-level subscription timeline with churn reason capture
- Core SaaS metric dashboard: MRR, ARR, LTV, ARPU, churn rate, and net revenue retention
- Voluntary vs. involuntary churn classification using dunning-stage signals

**Should-have (v1.1)**:
- Benchmark comparison against anonymised peer companies segmented by revenue band and vertical
- Plan-level and segment-level MRR breakdown with custom attribute filtering
- Expansion signal detection via product-usage data integration
- MRR movement alerts for anomalous changes (day-over-day deviation thresholds)
- Multi-currency support with configurable base currency and historical FX rate preservation

**Nice-to-have (backlog)**:
- Embedded analytics SDK for displaying subscription metrics inside customers' own products
- AI-generated natural-language MRR commentary for board reporting
- Automated churn risk scoring combining billing, usage, and support signals
- MRR forecast with scenario modelling and confidence intervals
- Dunning automation (retry sequences and personalised recovery offers) integrated with analytics
