# Subscription Analytics Platform

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> An open, AI-native subscription analytics platform that turns billing event streams into a trustworthy MRR ledger, cohort retention curves, and explainable churn analytics.

Subscription Analytics Platform ingests billing events from payment processors and CRMs, constructs a continuous MRR ledger, and exposes structured analytics across revenue movement, cohort behaviour, and churn classification. It is built for finance teams, founders, and revenue-operations leaders at SaaS and recurring-revenue businesses who currently rely on spreadsheets or general-purpose BI tools that were never designed to model recurring revenue.

---

## Why Subscription Analytics Platform?

- **Incumbents are proprietary, closed SaaS.** ChartMogul, Baremetrics, and ProfitWell (Paddle) are all proprietary platforms; there is no credible open-source alternative giving operators full control of their billing data and analytical pipeline.
- **Stripe-centric lock-in.** Baremetrics and ProfitWell are effectively Stripe- or Paddle-ecosystem products with limited support for Chargebee, Recurly, and legacy ERPs.
- **Weak involuntary-churn attribution.** Across the field, voluntary vs. involuntary churn classification and dunning-stage analytics are poorly developed, leaving teams unable to see which payment-failure recovery steps are actually working.
- **No product-usage signal integration.** Existing tools do not natively join billing events with product-usage data, so expansion-signal detection and proactive outreach are out of reach without a separate analytics stack.
- **Benchmark data is a trade secret.** ChartMogul's and ProfitWell's benchmark datasets are proprietary; an open initiative could establish transparent, auditable peer benchmarking methodology.

---

## Key Features

### MRR Ledger and Movement Analytics

- Continuous MRR ledger constructed from billing events with a five-movement waterfall: new, expansion, contraction, churn, and reactivation
- Core SaaS metric dashboard covering MRR, ARR, LTV, ARPU, churn rate, and net revenue retention
- Plan-level and segment-level MRR breakdown with custom attribute filtering
- Real-time MRR movement alerts for anomalous day-over-day deviations
- Multi-currency support with configurable base currency and historical FX rate preservation

### Cohort and Customer Analytics

- Cohort net-MRR-retention grid by signup month and plan, netting expansion and reactivation against contraction and churn
- Customer-level subscription timeline with plan changes, pauses, cancellations, and churn reason capture
- Trial-to-paid conversion analysis by cohort and acquisition channel
- Segmentation of MRR movements by plan, geography, customer attribute, or custom tag

### Churn Classification and Recovery

- Voluntary vs. involuntary churn classification using dunning-stage signals
- Dunning-stage analytics tracking which payment-failure recovery steps (retry 1, retry 2, email, SMS) recover what share of involuntary churn
- Optional dunning automation with retry sequences and personalised recovery offers integrated with analytics

### Benchmarking and Forecasting

- Benchmark comparison against anonymised peer companies segmented by revenue band and vertical
- MRR forecast with scenario modelling and confidence intervals
- Expansion signal detection via product-usage data integration

### Embedding and Distribution

- Embeddable dashboards for cohort and waterfall charts
- Embedded analytics SDK for displaying subscription metrics inside customers' own products
- REST API and CSV export for data portability

---

## AI-Native Advantage

AI capabilities target the precise gaps left by incumbents: automated churn risk scoring that combines billing patterns, product usage, and support interaction history; natural-language MRR commentary suitable for board reporting (for example, "MRR declined 3.2% this month primarily due to annual-plan churn in the SMB segment"); forecast modelling under multiple growth and churn scenarios with confidence intervals; and anomaly detection on daily MRR movements that flags data pipeline issues before monthly close.

---

## Tech Stack & Deployment

The platform is designed to be assembled from open-source components: dbt for SQL-based MRR ledger transformations; Apache Kafka or Redpanda for near-real-time event streaming; Snowflake, BigQuery, or Redshift as the analytical warehouse; Apache Superset or Metabase as the embeddable BI layer; Python (pandas, lifetimes) for cohort survival analysis and LTV projection; and Airflow or Prefect for scheduled MRR reconciliation jobs. Native connectors target Stripe, Chargebee, and Recurly as primary billing event sources, with Segment and Rudderstack for joining billing events to product-usage signals.

---

## Market Context

The SaaS analytics market is growing at roughly 10–15% CAGR, and over 80% of SaaS companies now use a dedicated reporting or analytics platform. Incumbent pricing ranges from free at all tiers (ProfitWell Metrics, with paid Retain add-on) to commercial SaaS plans (ChartMogul, Baremetrics, Chargebee) and percent-of-revenue billing-plus-analytics bundles (Stripe Billing). Primary buyers are SaaS finance teams, founders, and revenue-operations leaders who need defensible MRR reporting and cohort-level retention insight.

---

## Project Status

> This project is in the **research and specification phase**.  
> Contributions, feedback, and domain expertise are welcome.

---

## Contributing

We welcome contributions from developers, domain experts, and potential users.
See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

**Important:** All contributions must be your own original work or clearly attributed
open-source material with a compatible licence. Copyright infringement and licence
violations will not be tolerated and will result in immediate removal of the offending
contribution. If you are unsure whether a piece of code, text, or other material is
safe to contribute, open an issue and ask before submitting.

---

## Licence

Licence to be determined. See [discussion](#) for context.
