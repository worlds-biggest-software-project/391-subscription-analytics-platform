# Project 391 — Subscription Analytics Platform

**Date:** 2026-05-02

---

## 1. Problem Statement

Subscription businesses generate a continuous stream of revenue events — upgrades, downgrades, pauses, cancellations, and reactivations — yet most finance and product teams still rely on spreadsheet snapshots or general-purpose BI tools that were never designed to model recurring revenue. The result is a persistent blind spot: leadership cannot reliably explain why MRR moved in a given month, which cohorts are degrading fastest, or whether churn is voluntary (customers choosing to leave) or involuntary (payment failures). Without that granularity, retention interventions are reactive, pricing experiments lack rigour, and board-level forecasts carry hidden uncertainty.

---

## 2. Proposed Solution

A purpose-built subscription analytics platform that ingests billing events from payment processors and CRMs, constructs a continuous MRR ledger, and exposes structured analytics across three axes: revenue movement (new, expansion, contraction, churn, reactivation), cohort behaviour (net MRR retention curves by signup month and plan), and churn classification (voluntary vs. involuntary, with dunning-stage breakdowns). The platform would offer embeddable dashboards, alerting on anomalous MRR movements, and a benchmarking layer comparing key metrics against anonymised peers in the same revenue band and vertical.

---

## 3. Market Landscape

The SaaS analytics market is growing at roughly 10–15% CAGR, and over 80% of SaaS companies now use dedicated reporting or analytics platforms. Established players in the space include:

- **ChartMogul** — offers MRR segmentation, cohort net-MRR-retention charts, and benchmarking against 30,000-plus SaaS companies. Its cohort view tracks how much recurring revenue is retained in subsequent periods after a customer begins a subscription, netting expansion and reactivation against contraction and churn. ([ChartMogul Help Center](https://help.chartmogul.com/article/160-cohort-net-mrr-retention))
- **Baremetrics** — surfaces churn breakdown by plan and cohort, allowing operators to see that annual subscribers churn at half the rate of monthly plans, with revenue-retention cohorts showing which signup months produce the highest-LTV customers. ([Renlar](https://renlar.com/blog/subscription-analytics))
- **ProfitWell Metrics (Paddle)** — positions itself as the only enterprise-grade subscription analytics platform available at zero cost, covering MRR, churn, LTV, and revenue retention with bank-level accuracy. ([Swell](https://www.swell.is/content/subscription-analytics))
- **MCP Analytics** — a newer entrant focused on AI-driven revenue-trend analysis for subscription businesses. ([MCP Analytics](https://mcpanalytics.ai/articles/revenue-trend-analysis-subscription-businesses))

Gaps in the current landscape include weak involuntary-churn attribution, limited dunning-stage analytics, and poor integration with product-usage data that would enable expansion-signal detection.

---

## 4. Key Challenges

- **Data normalisation** — billing events from Stripe, Chargebee, Recurly, and legacy ERPs use different event schemas and timestamp conventions; unifying them without losing precision is non-trivial.
- **Voluntary vs. involuntary churn classification** — distinguishing a deliberate cancellation from a payment failure that was never recovered requires tracking the full dunning lifecycle, which most billing APIs expose inconsistently.
- **Cohort methodology agreement** — there is no universal standard for how to define a cohort start date (first charge, trial end, contract signature), which makes benchmark comparisons misleading if not carefully governed.
- **Real-time vs. batch trade-off** — finance teams expect end-of-month closes to match billing system records exactly; real-time dashboards introduce reconciliation risk if event streams and batch jobs are not carefully coordinated.
- **Multi-currency and multi-plan complexity** — SaaS businesses operating across currencies or with complex multi-seat pricing require normalisation logic that is error-prone and hard to audit.

---

## 5. Relevant Tools & Technologies

- **Stripe Billing API / Chargebee / Recurly** — primary billing event sources
- **dbt (data build tool)** — SQL-based transformation layer for constructing the MRR ledger from raw events
- **Apache Kafka / Redpanda** — event streaming for near-real-time MRR movement alerts
- **Snowflake / BigQuery / Redshift** — analytical data warehouses for cohort computations at scale
- **Apache Superset / Metabase** — open-source BI layers for embeddable cohort and waterfall charts
- **Python (pandas, lifetimes library)** — cohort survival analysis and LTV projection modelling
- **Retool / Observable** — internal dashboard tooling for finance and revenue-operations teams
- **Segment / Rudderstack** — customer data platforms for joining billing events with product-usage signals
- **Airflow / Prefect** — orchestration for scheduled MRR reconciliation jobs

---

## Sources

- [ChartMogul Help Center — Cohort: Net MRR Retention](https://help.chartmogul.com/article/160-cohort-net-mrr-retention)
- [Renlar — Best Subscription Analytics Software in 2026](https://renlar.com/blog/subscription-analytics)
- [Swell — Subscription Analytics Metrics That Matter for Recurring Revenue (2026 Guide)](https://www.swell.is/content/subscription-analytics)
- [MCP Analytics — Revenue Trend Analysis for Subscription Businesses](https://mcpanalytics.ai/articles/revenue-trend-analysis-subscription-businesses)
- [Julius AI — 8 Best SaaS Analytics Software of 2026](https://julius.ai/articles/saas-analytics-software)
- [Chartsy Blog — MRR Dashboard: How to Visualize Monthly Recurring Revenue (2026 Guide)](https://chartsy.app/blog/mrr-dashboard-visualize-monthly-recurring-revenue)
- [ReferralCandy — Subscription Analytics Ecommerce: The Complete 2026 Guide](https://www.referralcandy.com/blog/subscription-analytics-ecommerce-the-complete-2026-guide-to-data-driven-growth)
- [PowerDrill — Top 10 SaaS Analytics Tools in 2026](https://powerdrill.ai/blog/top-10-saas-analytics-tools)
