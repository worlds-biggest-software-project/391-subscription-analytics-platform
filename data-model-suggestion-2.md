# Data Model Suggestion 2: Hybrid Relational + JSONB

> Project: Subscription Analytics Platform · Created: 2026-05-29

## Philosophy

This model uses relational tables for high-query-rate entities (customers, subscriptions, MRR movements) while embedding configuration, plan catalogues, dunning sequences, cohort definitions, and benchmark data into JSONB columns on their parent records. The tenant record becomes a self-contained configuration document holding data source connections, plan definitions, segmentation rules, and forecast parameters, while the hot analytical path — billing events, MRR movements, and usage events — remains fully relational and partitioned for performance.

The approach recognises that subscription analytics has two distinct data access patterns: a write-heavy event ingestion pipeline that needs relational indexing for deduplication and processing, and a read-heavy configuration layer where tenant setup, plan catalogues, and benchmark definitions change infrequently. Embedding the latter into JSONB reduces table count and eliminates joins for configuration lookups while preserving relational rigour where query performance matters most.

**Best for:** Rapid MVP development where the team wants to iterate on tenant configuration, plan structures, and segmentation rules without schema migrations, while maintaining strong relational guarantees on the MRR ledger.

**Trade-offs:**
- (+) Fewer tables means simpler migrations and faster onboarding for new developers
- (+) Tenant configuration is a single atomic document — easy to clone, export, or version
- (+) New billing source fields or plan attributes added without ALTER TABLE
- (+) MRR movement and event queries remain fully relational with standard SQL
- (-) JSONB fields bypass referential integrity — plan_id references inside subscriptions must be validated in application code
- (-) Reporting on embedded fields (e.g., "all plans across all tenants with annual billing") requires JSONB operators
- (-) Larger row sizes for tenant records with many embedded entities

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ASC 606 / IFRS 15 | Revenue recognition mode stored in tenant config; MRR movements maintain ASC 606-compliant periodic recognition |
| CloudEvents v1.0 | Billing events use CloudEvents envelope for vendor-neutral ingestion |
| ISO 4217 | Currency codes in tenant base_currency and per-subscription currency |
| ISO 3166-1 | Country codes in customer records and benchmark segments |
| OAuth 2.0 | Billing source OAuth credentials referenced in tenant data_sources_json |
| PCI DSS v4.0 | Credential vault references only — no raw secrets in JSONB |
| GDPR / CCPA | Customer anonymisation via is_deleted flag preserving aggregate metrics |

---

## Core Tables

```sql
CREATE TABLE tenants (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL UNIQUE,
    base_currency   CHAR(3) NOT NULL DEFAULT 'USD',

    -- Embedded users
    users_json      JSONB NOT NULL DEFAULT '[]',
    -- [{
    --   "id": "uuid", "email": "...", "name": "...",
    --   "role": "owner|admin|analyst|viewer|api_service|embed_viewer",
    --   "is_active": true
    -- }]

    -- Billing source connections
    data_sources_json JSONB NOT NULL DEFAULT '[]',
    -- [{
    --   "id": "uuid", "name": "Stripe Production",
    --   "provider": "stripe|chargebee|recurly|zuora|paddle|braintree|maxio|custom_api|csv_import",
    --   "status": "active", "auth_method": "api_key|oauth2|webhook_only",
    --   "credentials_ref": "vault://...", "webhook_secret_ref": "vault://...",
    --   "last_sync_at": "2026-05-29T...", "sync_cursor": "...",
    --   "event_types": ["customer.*", "subscription.*", "invoice.*"],
    --   "config": {"sync_interval_minutes": 15, "backfill_from": "2024-01-01"}
    -- }]

    -- Plan catalogue
    plans_json      JSONB NOT NULL DEFAULT '[]',
    -- [{
    --   "id": "uuid", "data_source_id": "uuid", "external_id": "plan_pro",
    --   "name": "Pro Monthly", "billing_interval": "monthly",
    --   "billing_interval_months": 1,
    --   "amount_cents": 9900, "currency": "USD",
    --   "is_trial": false, "trial_days": null, "is_active": true
    -- }]

    -- Dunning configuration
    dunning_json    JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "enabled": true,
    --   "sequences": [{
    --     "step": 1, "type": "payment_retry", "delay_hours": 24
    --   }, {
    --     "step": 2, "type": "email", "delay_hours": 72,
    --     "template": "payment_failed_1"
    --   }, {
    --     "step": 3, "type": "payment_retry", "delay_hours": 120
    --   }, {
    --     "step": 4, "type": "sms", "delay_hours": 168
    --   }, {
    --     "step": 5, "type": "final_notice", "delay_hours": 240
    --   }],
    --   "recovery_offers": {
    --     "discount_enabled": true, "max_discount_percent": 20,
    --     "pause_enabled": true, "max_pause_days": 30
    --   }
    -- }

    -- Segmentation & cohort configuration
    segments_json   JSONB NOT NULL DEFAULT '[]',
    -- [{
    --   "key": "company_size", "label": "Company Size",
    --   "values": ["smb", "mid_market", "enterprise"],
    --   "source": "custom_attribute|plan|country|acquisition_channel"
    -- }]

    -- Forecast configuration
    forecasts_json  JSONB NOT NULL DEFAULT '[]',
    -- [{
    --   "id": "uuid", "name": "Q3 Base Case", "scenario": "base",
    --   "forecast_months": 12, "start_month": "2026-07-01",
    --   "assumptions": {
    --     "monthly_new_mrr_cents": 50000, "new_mrr_growth_rate": 0.05,
    --     "gross_churn_rate": 0.03, "expansion_rate": 0.02,
    --     "confidence_level": 0.90
    --   },
    --   "results": [{"month": "2026-07-01", "mrr_cents": 1500000, ...}]
    -- }]

    -- Platform configuration
    config_json     JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "plan": "growth", "max_customers": 10000,
    --   "fiscal_year_start_month": 1,
    --   "revenue_recognition": "asc_606|ifrs_15|cash_basis",
    --   "webhooks": [{"url": "...", "events": ["mrr.*"], "secret_ref": "vault://..."}],
    --   "embed": {"allowed_domains": ["app.example.com"], "theme": {...}},
    --   "alerts": {
    --     "mrr_deviation_threshold_pct": 5.0,
    --     "channels": ["email", "slack"],
    --     "slack_webhook_url": "..."
    --   }
    -- }

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_tenants_slug ON tenants(slug);
CREATE INDEX idx_tenants_data_sources ON tenants USING GIN (data_sources_json);
CREATE INDEX idx_tenants_plans ON tenants USING GIN (plans_json);
```

## Customers & Subscriptions

```sql
CREATE TABLE customers (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    data_source_id  UUID NOT NULL,  -- references tenants.data_sources_json[].id
    external_id     TEXT NOT NULL,
    name            TEXT,
    email           TEXT,
    company         TEXT,
    country         CHAR(2),
    status          TEXT NOT NULL DEFAULT 'active' CHECK (status IN (
        'lead', 'trialing', 'active', 'past_due', 'cancelled', 'churned', 'reactivated'
    )),
    acquisition_channel TEXT,
    first_seen_at   TIMESTAMPTZ NOT NULL,
    churned_at      TIMESTAMPTZ,
    churn_type      TEXT CHECK (churn_type IN ('voluntary', 'involuntary')),
    churn_reason    TEXT,
    tags            TEXT[] NOT NULL DEFAULT '{}',
    custom_attributes JSONB NOT NULL DEFAULT '{}',

    -- Embedded subscription history
    subscriptions_json JSONB NOT NULL DEFAULT '[]',
    -- [{
    --   "id": "uuid", "plan_id": "uuid", "external_id": "sub_xxx",
    --   "status": "active", "quantity": 1,
    --   "mrr_cents": 9900, "currency": "USD",
    --   "start_date": "2025-01-15", "trial_end_date": null,
    --   "current_period_start": "2026-05-15", "current_period_end": "2026-06-15",
    --   "cancelled_at": null, "cancellation_reason": null,
    --   "cancel_at_period_end": false,
    --   "discount_percent": null, "discount_amount_cents": null,
    --   "history": [
    --     {"date": "2025-01-15", "action": "created", "plan_id": "uuid", "mrr_cents": 4900},
    --     {"date": "2025-06-01", "action": "upgraded", "plan_id": "uuid", "mrr_cents": 9900},
    --     {"date": "2025-09-01", "action": "paused"},
    --     {"date": "2025-10-01", "action": "resumed"}
    --   ]
    -- }]

    -- Embedded dunning history
    dunning_json    JSONB NOT NULL DEFAULT '[]',
    -- [{
    --   "subscription_id": "uuid", "started_at": "2026-04-15T...",
    --   "steps": [
    --     {"sequence": 1, "type": "payment_retry", "at": "...", "outcome": "failed"},
    --     {"sequence": 2, "type": "email", "at": "...", "outcome": "pending"},
    --     {"sequence": 3, "type": "payment_retry", "at": "...", "outcome": "recovered",
    --      "recovered_amount_cents": 9900}
    --   ]
    -- }]

    -- Usage summary (aggregated from usage_events)
    usage_summary_json JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "last_active_at": "2026-05-28T...",
    --   "activity_score": 0.85,
    --   "feature_usage": {"api_calls": 1250, "dashboards_viewed": 42, "exports": 3},
    --   "trend": "increasing|stable|declining|inactive"
    -- }

    is_deleted      BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, data_source_id, external_id)
);

CREATE INDEX idx_customers_tenant ON customers(tenant_id);
CREATE INDEX idx_customers_status ON customers(tenant_id, status);
CREATE INDEX idx_customers_country ON customers(tenant_id, country);
CREATE INDEX idx_customers_tags ON customers USING GIN (tags);
CREATE INDEX idx_customers_custom_attrs ON customers USING GIN (custom_attributes);
CREATE INDEX idx_customers_subscriptions ON customers USING GIN (subscriptions_json);
```

## Billing Events & MRR Ledger

```sql
CREATE TABLE billing_events (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    data_source_id  UUID NOT NULL,
    customer_id     UUID REFERENCES customers(id),

    -- CloudEvents envelope
    ce_source       TEXT NOT NULL,
    ce_type         TEXT NOT NULL,
    ce_time         TIMESTAMPTZ NOT NULL,
    ce_specversion  TEXT NOT NULL DEFAULT '1.0',

    event_category  TEXT NOT NULL CHECK (event_category IN (
        'subscription_created', 'subscription_updated', 'subscription_cancelled',
        'subscription_paused', 'subscription_resumed', 'subscription_reactivated',
        'plan_changed', 'quantity_changed', 'discount_applied', 'discount_removed',
        'trial_started', 'trial_converted', 'trial_expired',
        'invoice_paid', 'invoice_failed', 'invoice_refunded',
        'payment_failed', 'payment_recovered', 'payment_refunded',
        'dunning_started', 'dunning_step', 'dunning_recovered', 'dunning_exhausted',
        'customer_created', 'customer_updated', 'customer_deleted'
    )),

    -- Processed MRR impact (embedded rather than separate table)
    mrr_impact_json JSONB,
    -- {
    --   "movement_type": "expansion",
    --   "amount_cents": 5000, "currency": "USD",
    --   "amount_base_cents": 5000, "fx_rate": 1.0,
    --   "previous_mrr_cents": 4900, "new_mrr_cents": 9900,
    --   "plan_id": "uuid", "previous_plan_id": "uuid",
    --   "churn_type": null, "dunning_stage": null,
    --   "segment_tags": ["enterprise", "annual"]
    -- }

    raw_payload     JSONB NOT NULL,
    is_processed    BOOLEAN NOT NULL DEFAULT false,
    processed_at    TIMESTAMPTZ,
    idempotency_key TEXT NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, data_source_id, idempotency_key)
) PARTITION BY RANGE (ce_time);

CREATE INDEX idx_billing_events_tenant_time ON billing_events(tenant_id, ce_time);
CREATE INDEX idx_billing_events_customer ON billing_events(customer_id);
CREATE INDEX idx_billing_events_category ON billing_events(tenant_id, event_category);
CREATE INDEX idx_billing_events_unprocessed ON billing_events(tenant_id) WHERE NOT is_processed;
CREATE INDEX idx_billing_events_mrr ON billing_events USING GIN (mrr_impact_json) WHERE mrr_impact_json IS NOT NULL;

CREATE TABLE mrr_snapshots (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    snapshot_date   DATE NOT NULL,

    -- Daily MRR state
    total_mrr_cents     BIGINT NOT NULL,
    total_customers     INT NOT NULL,
    total_subscriptions INT NOT NULL,

    -- Movement waterfall for this day
    movements_json  JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "new_cents": 15000, "new_count": 3,
    --   "expansion_cents": 8000, "expansion_count": 5,
    --   "contraction_cents": -2000, "contraction_count": 1,
    --   "churn_cents": -9900, "churn_count": 1,
    --   "churn_voluntary_cents": -4900, "churn_involuntary_cents": -5000,
    --   "reactivation_cents": 4900, "reactivation_count": 1,
    --   "net_change_cents": 16000
    -- }

    -- Per-plan breakdown
    plan_breakdown_json JSONB NOT NULL DEFAULT '[]',
    -- [{"plan_id": "uuid", "name": "Pro", "mrr_cents": 495000, "customers": 50}]

    -- Per-segment breakdown
    segment_breakdown_json JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "country": {"US": {"mrr_cents": 800000}, "GB": {"mrr_cents": 200000}},
    --   "company_size": {"smb": {"mrr_cents": 300000}, "enterprise": {"mrr_cents": 700000}}
    -- }

    -- Key metrics
    metrics_json    JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "arr_cents": 12000000, "arpu_cents": 9900,
    --   "gross_churn_rate": 0.028, "net_churn_rate": -0.005,
    --   "logo_churn_rate": 0.02, "net_revenue_retention": 1.05,
    --   "gross_revenue_retention": 0.972,
    --   "quick_ratio": 3.5,
    --   "trial_conversion_rate": 0.42
    -- }

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, snapshot_date)
) PARTITION BY RANGE (snapshot_date);

CREATE INDEX idx_mrr_snapshots_tenant ON mrr_snapshots(tenant_id, snapshot_date);
```

## Benchmarks & Usage

```sql
CREATE TABLE benchmarks (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    period_month    DATE NOT NULL,
    revenue_band    TEXT NOT NULL CHECK (revenue_band IN (
        'pre_revenue', '0_10k', '10k_50k', '50k_100k', '100k_500k',
        '500k_1m', '1m_5m', '5m_10m', '10m_50m', '50m_plus'
    )),
    vertical        TEXT NOT NULL CHECK (vertical IN (
        'horizontal_saas', 'vertical_saas', 'developer_tools', 'fintech',
        'healthtech', 'edtech', 'ecommerce', 'martech', 'hrtech', 'other'
    )),
    metrics_json    JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "mrr_growth_rate": {"p10": 0.01, "p25": 0.03, "p50": 0.06, "p75": 0.10, "p90": 0.18},
    --   "gross_churn_rate": {"p10": 0.01, "p25": 0.02, "p50": 0.035, "p75": 0.06, "p90": 0.10},
    --   "net_revenue_retention": {"p10": 0.85, "p25": 0.95, "p50": 1.05, "p75": 1.15, "p90": 1.30},
    --   "ltv_months": {"p10": 12, "p25": 18, "p50": 30, "p75": 48, "p90": 72},
    --   "arpu_cents": {"p10": 2900, "p25": 4900, "p50": 9900, "p75": 24900, "p90": 99900},
    --   "trial_conversion_rate": {"p10": 0.10, "p25": 0.20, "p50": 0.35, "p75": 0.50, "p90": 0.65},
    --   "quick_ratio": {"p10": 1.2, "p25": 2.0, "p50": 3.0, "p75": 4.5, "p90": 7.0}
    -- }
    sample_size     INT NOT NULL,
    is_published    BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (period_month, revenue_band, vertical)
);

CREATE INDEX idx_benchmarks_lookup ON benchmarks(revenue_band, vertical, period_month);

CREATE TABLE usage_events (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    customer_id     UUID NOT NULL REFERENCES customers(id),
    event_name      TEXT NOT NULL,
    event_date      DATE NOT NULL,
    event_count     INT NOT NULL DEFAULT 1,
    properties      JSONB NOT NULL DEFAULT '{}',
    source          TEXT NOT NULL CHECK (source IN ('segment', 'rudderstack', 'custom_api', 'csv_import')),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (event_date);

CREATE INDEX idx_usage_events_tenant_date ON usage_events(tenant_id, event_date);
CREATE INDEX idx_usage_events_customer ON usage_events(customer_id);
CREATE INDEX idx_usage_events_name ON usage_events(tenant_id, event_name);
```

## AI & Audit

```sql
CREATE TABLE ai_suggestions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    suggestion_type TEXT NOT NULL CHECK (suggestion_type IN (
        'churn_risk', 'expansion_signal', 'anomaly_detected', 'mrr_commentary',
        'dunning_optimization', 'cohort_insight', 'benchmark_comparison',
        'forecast_adjustment', 'data_quality_issue', 'segment_recommendation'
    )),
    entity_type     TEXT,
    entity_id       UUID,
    severity        TEXT NOT NULL CHECK (severity IN ('info', 'warning', 'critical')),
    title           TEXT NOT NULL,
    description     TEXT NOT NULL,
    evidence        JSONB NOT NULL DEFAULT '{}',
    recommended_action TEXT,
    status          TEXT NOT NULL DEFAULT 'pending' CHECK (status IN (
        'pending', 'accepted', 'dismissed', 'expired', 'auto_applied'
    )),
    confidence      NUMERIC(5,4),
    model_version   TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    resolved_at     TIMESTAMPTZ
);

CREATE INDEX idx_ai_suggestions_tenant ON ai_suggestions(tenant_id);
CREATE INDEX idx_ai_suggestions_type ON ai_suggestions(tenant_id, suggestion_type);
CREATE INDEX idx_ai_suggestions_pending ON ai_suggestions(tenant_id) WHERE status = 'pending';

CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    actor_type      TEXT NOT NULL CHECK (actor_type IN ('user', 'system', 'api_key', 'webhook', 'ai', 'scheduler')),
    actor_id        TEXT,
    action          TEXT NOT NULL,
    resource_type   TEXT NOT NULL,
    resource_id     UUID,
    changes         JSONB,
    ip_address      INET,
    user_agent      TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (created_at);

CREATE INDEX idx_audit_log_tenant ON audit_log(tenant_id, created_at);
CREATE INDEX idx_audit_log_resource ON audit_log(resource_type, resource_id);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Configuration | 1 | tenants (embeds users, data sources, plans, dunning config, segments, forecasts) |
| Customers | 1 | customers (embeds subscriptions, dunning history, usage summary) |
| Events & MRR | 2 | billing_events (partitioned, embeds mrr_impact), mrr_snapshots (partitioned) |
| Benchmarks & Usage | 2 | benchmarks (all metrics in JSONB), usage_events (partitioned) |
| AI & Audit | 2 | ai_suggestions, audit_log (partitioned) |
| **Total** | **8** | 4 partitioned tables |

---

## Key Design Decisions

1. **MRR impact embedded on billing events** — instead of a separate mrr_movements table, each billing event carries its MRR impact as mrr_impact_json. This collocates cause and effect: a single query shows the event and its MRR consequence without a join.

2. **Daily MRR snapshots replace per-movement queries** — mrr_snapshots pre-computes the waterfall, plan breakdown, segment breakdown, and key metrics daily. Dashboard queries hit this table directly rather than aggregating millions of movement rows.

3. **Subscriptions embedded in customers** — a customer's subscription history (including plan changes, pauses, cancellations) is a single JSONB array on the customer record. The customer timeline view is a single-row fetch. Active subscription MRR is still queryable via `subscriptions_json @> '[{"status":"active"}]'`.

4. **Dunning sequences configured on tenant, tracked on customer** — the dunning step definitions live in tenants.dunning_json (configuration), while actual dunning attempts per customer are in customers.dunning_json (execution state). This separates template from instance.

5. **Benchmark metrics consolidated** — all percentile distributions for a given revenue band / vertical / month are in a single JSONB column, reducing the benchmarks table from dozens of rows per combination to one row. Ideal for the comparison dashboard that shows all metrics side by side.

6. **Usage summary pre-aggregated on customer** — customers.usage_summary_json provides the activity score and feature usage for expansion signal detection without scanning usage_events. The detailed events remain in the partitioned table for deeper analysis.

7. **Tenant as configuration document** — cloning a tenant's setup for a new environment is a single row copy. Plan catalogue, segmentation rules, forecast parameters, and dunning config are all atomic.

8. **Billing events remain relational** — despite the hybrid approach, billing_events keeps event_category as a CHECK-constrained column for fast filtered queries and the CloudEvents envelope as relational fields, because event ingestion is the hottest write path.
