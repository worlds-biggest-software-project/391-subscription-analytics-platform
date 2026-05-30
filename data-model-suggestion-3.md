# Data Model Suggestion 3: Event-Sourced / Audit-First

> Project: Subscription Analytics Platform · Created: 2026-05-29

## Philosophy

This model treats every state change as an immutable event in a single append-only event store. The MRR ledger is not a table — it is a projection computed by replaying billing and subscription events through a five-movement classifier. Cohort retention, dunning recovery rates, and customer timelines are all derived from the event stream rather than maintained as mutable state. Read models (materialised projections) serve the dashboard, forecast, and benchmark queries.

For subscription analytics, event sourcing is a natural fit: billing providers already emit event streams (customer.subscription.updated, invoice.payment_failed), and the core analytical value is in understanding sequences — what happened between trial start and conversion, which dunning step recovered the payment, how MRR moved over time. An event-sourced model preserves the complete temporal history needed for "what was MRR on date X?" queries, cohort survival analysis, and AI-driven churn risk scoring based on behavioural patterns.

**Best for:** Platforms requiring full temporal audit trails, "as-of" reporting for board decks and investor updates, AI/ML feature engineering from event sequences, and regulatory compliance with ASC 606/IFRS 15 revenue recognition that demands traceable revenue allocation.

**Trade-offs:**
- (+) Complete temporal history enables any retroactive analysis without data loss
- (+) "As-of" queries trivially answered: replay events up to any point in time
- (+) AI churn risk models can train on full behavioural sequences
- (+) New metrics or cohort definitions added by creating new projections without backfilling
- (-) Read models must be maintained and kept in sync with the event store
- (-) Simple lookups ("current MRR") require read models rather than direct queries
- (-) Event replay for large tenants can be slow without snapshots and checkpoints
- (-) More complex to reason about for developers unfamiliar with CQRS

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ASC 606 / IFRS 15 | Revenue recognition is a projection: events replay to allocate revenue to performance obligations, enabling retroactive recalculation when recognition rules change |
| CloudEvents v1.0 | Event store uses CloudEvents envelope (ce_source, ce_type, ce_time, ce_specversion) as native event format |
| AsyncAPI 3.x | Event taxonomy documented as AsyncAPI channels for external consumers of the event stream |
| ISO 4217 | Currency codes on billing events and MRR projections |
| ISO 3166-1 | Country codes on customer events for geographic cohort projections |
| GDPR / CCPA | Crypto-shredding: customer PII encrypted with per-customer key; deletion destroys key, events become unreadable while aggregates remain intact |

---

## Event Infrastructure

```sql
CREATE TABLE event_store (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_type     TEXT NOT NULL CHECK (stream_type IN (
        'tenant', 'user', 'data_source', 'plan',
        'customer', 'subscription', 'billing_event',
        'dunning', 'cohort', 'benchmark', 'usage',
        'forecast', 'ai', 'config'
    )),
    stream_id       UUID NOT NULL,
    version         BIGINT NOT NULL,
    event_type      TEXT NOT NULL,
    actor_type      TEXT NOT NULL CHECK (actor_type IN (
        'user', 'system', 'api_key', 'webhook', 'ai',
        'billing_provider', 'scheduler', 'event_processor', 'projection_engine'
    )),
    actor_id        TEXT,
    tenant_id       UUID,

    -- CloudEvents envelope
    ce_source       TEXT NOT NULL,
    ce_type         TEXT NOT NULL,
    ce_time         TIMESTAMPTZ NOT NULL,
    ce_specversion  TEXT NOT NULL DEFAULT '1.0',

    data            JSONB NOT NULL,
    metadata        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (stream_type, stream_id, version)
) PARTITION BY RANGE (ce_time);

CREATE INDEX idx_events_stream ON event_store(stream_type, stream_id, version);
CREATE INDEX idx_events_tenant ON event_store(tenant_id, ce_time);
CREATE INDEX idx_events_type ON event_store(event_type, ce_time);

CREATE TABLE stream_snapshots (
    stream_type     TEXT NOT NULL,
    stream_id       UUID NOT NULL,
    version         BIGINT NOT NULL,
    state           JSONB NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (stream_type, stream_id, version)
);

CREATE TABLE projection_checkpoints (
    projection_name TEXT NOT NULL,
    partition_key   TEXT NOT NULL DEFAULT '_global',
    last_event_id   UUID NOT NULL,
    last_event_time TIMESTAMPTZ NOT NULL,
    state           JSONB NOT NULL DEFAULT '{}',
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (projection_name, partition_key)
);
```

## Event Taxonomy

### Tenant & Configuration Events
- `tenant_created` — {name, slug, base_currency, plan, revenue_recognition}
- `tenant_config_updated` — {field, old_value, new_value}
- `user_added` — {user_id, email, role}
- `user_role_changed` — {user_id, old_role, new_role}
- `user_removed` — {user_id}

### Data Source Events
- `data_source_connected` — {provider, auth_method, event_types, config}
- `data_source_sync_started` — {cursor}
- `data_source_sync_completed` — {events_ingested, duration_ms}
- `data_source_sync_failed` — {error, cursor}
- `data_source_disconnected` — {reason}

### Plan Events
- `plan_imported` — {external_id, name, billing_interval, amount_cents, currency}
- `plan_updated` — {field, old_value, new_value}
- `plan_deprecated` — {replacement_plan_id}

### Customer Events
- `customer_created` — {external_id, name, email, country, acquisition_channel}
- `customer_updated` — {fields_changed}
- `customer_tagged` — {tags_added, tags_removed}
- `customer_attribute_set` — {key, value}
- `customer_deleted` — {gdpr_request: true}  (triggers crypto-shredding)

### Subscription Events
- `subscription_created` — {external_id, plan_id, quantity, mrr_cents, currency, start_date, trial_end_date}
- `subscription_trial_started` — {trial_days, trial_end_date}
- `subscription_trial_converted` — {plan_id, mrr_cents}
- `subscription_trial_expired` — {}
- `subscription_plan_changed` — {old_plan_id, new_plan_id, old_mrr_cents, new_mrr_cents}
- `subscription_quantity_changed` — {old_quantity, new_quantity, old_mrr_cents, new_mrr_cents}
- `subscription_discount_applied` — {discount_percent, discount_amount_cents}
- `subscription_discount_removed` — {}
- `subscription_paused` — {resume_at}
- `subscription_resumed` — {}
- `subscription_cancelled` — {reason, cancel_at_period_end, churn_type: voluntary}
- `subscription_expired` — {}
- `subscription_reactivated` — {plan_id, mrr_cents}

### Billing Event Events (external events ingested from providers)
- `billing_event_ingested` — {ce_source, ce_type, event_category, raw_payload, idempotency_key}
- `billing_event_processed` — {mrr_movement_type, amount_cents, currency, fx_rate, amount_base_cents}
- `billing_event_skipped` — {reason: duplicate|irrelevant|malformed}

### MRR Movement Events (derived from subscription events)
- `mrr_new` — {customer_id, subscription_id, amount_cents, plan_id}
- `mrr_expansion` — {customer_id, subscription_id, amount_cents, previous_mrr_cents, new_mrr_cents}
- `mrr_contraction` — {customer_id, subscription_id, amount_cents, previous_mrr_cents, new_mrr_cents}
- `mrr_churn` — {customer_id, subscription_id, amount_cents, churn_type, churn_reason, dunning_stage}
- `mrr_reactivation` — {customer_id, subscription_id, amount_cents, months_since_churn}

### Dunning Events
- `dunning_started` — {customer_id, subscription_id, failed_amount_cents}
- `dunning_step_executed` — {sequence, step_type, outcome}
- `dunning_recovered` — {sequence, recovered_amount_cents, recovery_step_type}
- `dunning_exhausted` — {total_steps, final_amount_cents}
- `dunning_offer_presented` — {offer_type, discount_percent, pause_days}
- `dunning_offer_accepted` — {offer_type}
- `dunning_offer_declined` — {}

### Usage Events
- `usage_event_ingested` — {customer_id, event_name, event_count, source, properties}
- `usage_milestone_reached` — {customer_id, milestone, event_name, threshold}
- `usage_trend_detected` — {customer_id, trend: increasing|declining|inactive, period_days}

### Forecast & Benchmark Events
- `forecast_created` — {name, scenario, assumptions, forecast_months}
- `forecast_results_computed` — {results: [{month, mrr_cents, lower_bound, upper_bound}]}
- `benchmark_period_computed` — {period_month, revenue_band, vertical, metrics, sample_size}
- `benchmark_published` — {period_month}

### AI Events
- `ai_suggestion_generated` — {suggestion_type, entity_type, entity_id, title, description, evidence, confidence}
- `ai_suggestion_accepted` — {suggestion_id}
- `ai_suggestion_dismissed` — {suggestion_id, reason}
- `ai_churn_risk_scored` — {customer_id, risk_score, risk_factors, model_version}
- `ai_expansion_signal_detected` — {customer_id, signal_type, confidence}
- `ai_mrr_commentary_generated` — {period, commentary_text, key_drivers}
- `ai_anomaly_detected` — {metric, expected_value, actual_value, deviation_pct}

---

## Read Models

```sql
CREATE TABLE rm_mrr_dashboard (
    tenant_id       UUID NOT NULL,
    snapshot_date   DATE NOT NULL,

    -- Current state
    total_mrr_cents     BIGINT NOT NULL,
    total_arr_cents     BIGINT NOT NULL,
    total_customers     INT NOT NULL,
    total_subscriptions INT NOT NULL,
    arpu_cents          BIGINT NOT NULL,

    -- Daily waterfall
    new_cents           BIGINT NOT NULL DEFAULT 0,
    new_count           INT NOT NULL DEFAULT 0,
    expansion_cents     BIGINT NOT NULL DEFAULT 0,
    expansion_count     INT NOT NULL DEFAULT 0,
    contraction_cents   BIGINT NOT NULL DEFAULT 0,
    contraction_count   INT NOT NULL DEFAULT 0,
    churn_cents         BIGINT NOT NULL DEFAULT 0,
    churn_count         INT NOT NULL DEFAULT 0,
    churn_voluntary_cents   BIGINT NOT NULL DEFAULT 0,
    churn_involuntary_cents BIGINT NOT NULL DEFAULT 0,
    reactivation_cents  BIGINT NOT NULL DEFAULT 0,
    reactivation_count  INT NOT NULL DEFAULT 0,
    net_change_cents    BIGINT NOT NULL DEFAULT 0,

    -- Rates
    gross_churn_rate    NUMERIC(8,6),
    net_churn_rate      NUMERIC(8,6),
    logo_churn_rate     NUMERIC(8,6),
    net_revenue_retention NUMERIC(8,6),
    gross_revenue_retention NUMERIC(8,6),
    quick_ratio         NUMERIC(8,4),
    trial_conversion_rate NUMERIC(8,6),

    -- Breakdowns
    plan_breakdown      JSONB NOT NULL DEFAULT '[]',
    segment_breakdown   JSONB NOT NULL DEFAULT '{}',
    currency_breakdown  JSONB NOT NULL DEFAULT '{}',

    last_event_id   UUID NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (tenant_id, snapshot_date)
) PARTITION BY RANGE (snapshot_date);

CREATE TABLE rm_cohort_retention (
    tenant_id       UUID NOT NULL,
    cohort_month    DATE NOT NULL,
    plan_id         UUID,
    segment_key     TEXT,
    segment_value   TEXT,

    -- Cohort initial state
    initial_customers   INT NOT NULL,
    initial_mrr_cents   BIGINT NOT NULL,

    -- Period-by-period retention
    periods         JSONB NOT NULL DEFAULT '[]',
    -- [{
    --   "period": 0, "month": "2025-01-01",
    --   "active_customers": 100, "mrr_cents": 990000,
    --   "expansion_cents": 0, "contraction_cents": 0,
    --   "churn_cents": 0, "reactivation_cents": 0,
    --   "gross_retention_pct": 1.0, "net_retention_pct": 1.0
    -- }, {
    --   "period": 1, "month": "2025-02-01",
    --   "active_customers": 92, "mrr_cents": 960000,
    --   "expansion_cents": 25000, "contraction_cents": -5000,
    --   "churn_cents": -50000, "reactivation_cents": 0,
    --   "gross_retention_pct": 0.944, "net_retention_pct": 0.970
    -- }]

    last_event_id   UUID NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (tenant_id, cohort_month, plan_id, segment_key, segment_value)
);

CREATE TABLE rm_customer_timeline (
    tenant_id       UUID NOT NULL,
    customer_id     UUID NOT NULL,
    external_id     TEXT NOT NULL,
    name            TEXT,
    email           TEXT,
    company         TEXT,
    country         CHAR(2),
    status          TEXT NOT NULL,
    acquisition_channel TEXT,
    first_seen_at   TIMESTAMPTZ NOT NULL,
    tags            TEXT[] NOT NULL DEFAULT '{}',
    custom_attributes JSONB NOT NULL DEFAULT '{}',

    -- Current subscription state
    active_subscriptions JSONB NOT NULL DEFAULT '[]',
    -- [{
    --   "subscription_id": "uuid", "plan_id": "uuid", "plan_name": "Pro",
    --   "mrr_cents": 9900, "currency": "USD", "status": "active",
    --   "start_date": "2025-01-15", "current_period_end": "2026-06-15"
    -- }]

    total_mrr_cents     BIGINT NOT NULL DEFAULT 0,
    lifetime_value_cents BIGINT NOT NULL DEFAULT 0,

    -- Churn risk
    churn_risk_score    NUMERIC(5,4),
    churn_risk_factors  JSONB,
    usage_trend         TEXT,  -- increasing|stable|declining|inactive

    -- Complete event timeline (most recent N events)
    timeline        JSONB NOT NULL DEFAULT '[]',
    -- [{
    --   "at": "2026-05-28T...", "type": "subscription_plan_changed",
    --   "summary": "Upgraded from Starter to Pro (+$50/mo)",
    --   "mrr_impact_cents": 5000
    -- }]

    -- Dunning state
    dunning_active  BOOLEAN NOT NULL DEFAULT false,
    dunning_json    JSONB,

    last_event_id   UUID NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (tenant_id, customer_id)
);

CREATE INDEX idx_rm_customer_status ON rm_customer_timeline(tenant_id, status);
CREATE INDEX idx_rm_customer_churn_risk ON rm_customer_timeline(tenant_id, churn_risk_score DESC NULLS LAST);
CREATE INDEX idx_rm_customer_tags ON rm_customer_timeline USING GIN (tags);

CREATE TABLE rm_dunning_analytics (
    tenant_id       UUID NOT NULL,
    period_month    DATE NOT NULL,

    -- Overall dunning metrics
    total_dunning_started   INT NOT NULL DEFAULT 0,
    total_recovered         INT NOT NULL DEFAULT 0,
    total_exhausted         INT NOT NULL DEFAULT 0,
    recovery_rate           NUMERIC(8,6),
    total_at_risk_cents     BIGINT NOT NULL DEFAULT 0,
    total_recovered_cents   BIGINT NOT NULL DEFAULT 0,

    -- Per-step breakdown
    step_breakdown  JSONB NOT NULL DEFAULT '[]',
    -- [{
    --   "sequence": 1, "step_type": "payment_retry",
    --   "attempted": 200, "recovered": 120, "recovery_rate": 0.60,
    --   "recovered_cents": 1188000, "avg_recovery_time_hours": 24
    -- }, {
    --   "sequence": 2, "step_type": "email",
    --   "attempted": 80, "recovered": 24, "recovery_rate": 0.30,
    --   "recovered_cents": 237600, "avg_recovery_time_hours": 96
    -- }]

    -- Offer analysis
    offer_breakdown JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "discount": {"offered": 30, "accepted": 12, "acceptance_rate": 0.40, "retained_cents": 95040},
    --   "pause": {"offered": 20, "accepted": 8, "acceptance_rate": 0.40, "retained_cents": 63360}
    -- }

    -- Involuntary churn detail
    involuntary_churn_cents BIGINT NOT NULL DEFAULT 0,
    involuntary_churn_count INT NOT NULL DEFAULT 0,

    last_event_id   UUID NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (tenant_id, period_month)
);

CREATE TABLE rm_expansion_signals (
    tenant_id       UUID NOT NULL,
    customer_id     UUID NOT NULL,

    -- Current expansion signals
    expansion_score     NUMERIC(5,4),
    signals         JSONB NOT NULL DEFAULT '[]',
    -- [{
    --   "type": "usage_milestone", "event_name": "api_calls",
    --   "threshold": 10000, "current_value": 12500,
    --   "detected_at": "2026-05-28T...", "confidence": 0.85
    -- }, {
    --   "type": "feature_adoption", "feature": "team_seats",
    --   "adoption_rate": 0.90, "detected_at": "2026-05-25T...", "confidence": 0.72
    -- }]

    -- Current subscription context
    current_plan_id     UUID,
    current_mrr_cents   BIGINT NOT NULL DEFAULT 0,
    months_on_plan      INT,
    usage_percentile    NUMERIC(5,4),

    -- AI recommendation
    recommended_action  TEXT,
    recommended_plan_id UUID,
    estimated_expansion_cents BIGINT,

    last_event_id   UUID NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (tenant_id, customer_id)
);

CREATE INDEX idx_rm_expansion_score ON rm_expansion_signals(tenant_id, expansion_score DESC NULLS LAST);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Infrastructure | 3 | event_store (partitioned), stream_snapshots, projection_checkpoints |
| Read Models | 5 | rm_mrr_dashboard (partitioned), rm_cohort_retention, rm_customer_timeline, rm_dunning_analytics, rm_expansion_signals |
| **Total** | **8** | 2 partitioned tables |

---

## Key Design Decisions

1. **MRR as a projection, not a table** — the MRR waterfall is computed by replaying mrr_new/mrr_expansion/mrr_contraction/mrr_churn/mrr_reactivation events through the rm_mrr_dashboard projection. This means any retroactive correction (reclassifying a movement, fixing an FX rate) replays from the corrected event forward rather than updating mutable rows.

2. **Billing events as events about events** — external billing provider webhooks are ingested as `billing_event_ingested` events, then processed into subscription lifecycle events and MRR movement events. This two-stage design separates raw ingestion from normalised analytics.

3. **Dunning analytics as a dedicated read model** — rm_dunning_analytics materialises per-step recovery rates, per-offer acceptance rates, and involuntary churn totals. This directly serves the platform's key differentiator: showing which dunning steps actually recover revenue.

4. **Expansion signals as a read model** — rm_expansion_signals combines usage milestones, feature adoption, and AI confidence scores into a per-customer view. The projection listens to both usage_milestone_reached and ai_expansion_signal_detected events.

5. **Cohort retention as event replay** — rm_cohort_retention periods are rebuilt from subscription and MRR movement events. Adding a new segmentation dimension (e.g., cohort by acquisition channel) means creating a new projection that replays the same events — no backfill migration needed.

6. **Customer timeline as a read model** — rm_customer_timeline provides the single-customer drill-down view with full event history, current subscriptions, churn risk score, and dunning state. All derived from events; the customer has no mutable "master record."

7. **Crypto-shredding for GDPR** — customer PII in events is encrypted with a per-customer key. Deletion request destroys the key, making all historical events for that customer unreadable while preserving aggregate projections (MRR totals, cohort counts) unaffected.

8. **Forecast and benchmark events** — forecasts and benchmark computations are themselves events, enabling audit of when projections were computed, with what assumptions, and how they changed over time. Board-deck MRR numbers are traceable to the exact event state they were derived from.

9. **Usage milestone events for AI training** — usage_milestone_reached and usage_trend_detected events provide structured features for churn risk and expansion scoring models, without requiring the ML pipeline to scan raw usage_event_ingested events.

10. **Stream types cover the full SaaS lifecycle** — 14 stream types from tenant configuration through customer lifecycle, billing, dunning, usage, forecasting, and benchmarking ensure every state change is captured for temporal queries and AI feature engineering.
