# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: Subscription Analytics Platform · Created: 2026-05-29

## Philosophy

This model treats every domain concept as a first-class relational entity with explicit foreign keys and referential integrity. The MRR ledger is constructed from normalised billing events through a five-movement waterfall (new, expansion, contraction, churn, reactivation), with each movement stored as a discrete row in a dedicated table. Cohort analysis, customer segmentation, and benchmark comparisons all query across normalised joins rather than pre-computed aggregates.

The approach mirrors how ChartMogul and Baremetrics conceptually organise subscription data — customers, subscriptions, plans, invoices, and events as separate entities — but makes the relationships explicit in the schema rather than implicit in application logic. Multi-currency support uses a separate FX rate table preserving historical rates for accurate MRR normalisation across reporting periods.

**Best for:** Finance teams requiring auditable MRR calculations with full referential integrity, multi-source billing ingestion, and regulatory alignment with ASC 606/IFRS 15 revenue recognition.

**Trade-offs:**
- (+) Full referential integrity ensures data consistency across MRR movements, cohorts, and customer records
- (+) Standard SQL queries for all analytics — no JSONB operators or event replay required
- (+) Clear audit trail from billing event to MRR movement to metric
- (-) Higher table count increases migration complexity
- (-) Cohort and benchmark queries involve multi-table joins that may require materialised views at scale
- (-) Schema changes needed when adding new billing source fields or metric types

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ASC 606 / IFRS 15 | Revenue recognition alignment: subscriptions track performance obligation period; MRR movements recognise revenue on periodic delivery basis |
| CloudEvents v1.0 | Billing events stored with CloudEvents envelope attributes (ce_source, ce_type, ce_time) for vendor-neutral ingestion |
| AsyncAPI 3.x | Event taxonomy for webhook streams normalised into billing_events table |
| OpenAPI 3.1 | REST API surface documented per OAS 3.1; JSON Schema 2020-12 for metric definitions |
| ISO 4217 | Currency codes for multi-currency MRR normalisation |
| ISO 3166-1 | Country codes for geographic segmentation and benchmark cohorts |
| OAuth 2.0 / OIDC | API authentication and billing source connector authorisation |
| PCI DSS v4.0 | Cardholder data excluded at ingestion; only tokenised references stored |
| GDPR / CCPA | Customer PII deletion cascades through anonymisation without corrupting aggregate metrics |

---

## Entity Management

```sql
CREATE TABLE tenants (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL UNIQUE,
    base_currency   CHAR(3) NOT NULL DEFAULT 'USD',  -- ISO 4217
    plan            TEXT NOT NULL CHECK (plan IN ('free', 'starter', 'growth', 'enterprise')),
    max_customers   INT,
    max_data_sources INT NOT NULL DEFAULT 3,
    fiscal_year_start_month SMALLINT NOT NULL DEFAULT 1 CHECK (fiscal_year_start_month BETWEEN 1 AND 12),
    revenue_recognition TEXT NOT NULL DEFAULT 'asc_606' CHECK (revenue_recognition IN ('asc_606', 'ifrs_15', 'cash_basis')),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    email           TEXT NOT NULL,
    name            TEXT NOT NULL,
    role            TEXT NOT NULL CHECK (role IN ('owner', 'admin', 'analyst', 'viewer', 'api_service', 'embed_viewer')),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, email)
);

CREATE INDEX idx_users_tenant ON users(tenant_id);
```

## Billing Sources & Plans

```sql
CREATE TABLE data_sources (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    name            TEXT NOT NULL,
    provider        TEXT NOT NULL CHECK (provider IN (
        'stripe', 'chargebee', 'recurly', 'zuora', 'paddle',
        'braintree', 'maxio', 'custom_api', 'csv_import'
    )),
    status          TEXT NOT NULL DEFAULT 'pending' CHECK (status IN (
        'pending', 'connected', 'syncing', 'active', 'error', 'disconnected'
    )),
    auth_method     TEXT NOT NULL CHECK (auth_method IN ('api_key', 'oauth2', 'webhook_only')),
    credentials_ref TEXT,  -- vault reference, never raw credentials
    webhook_secret_ref TEXT,
    last_sync_at    TIMESTAMPTZ,
    sync_cursor     TEXT,  -- provider-specific pagination cursor
    event_types     TEXT[] NOT NULL DEFAULT '{}',  -- subscribed event types
    config          JSONB NOT NULL DEFAULT '{}',
    -- config example: {"sync_interval_minutes": 15, "backfill_from": "2024-01-01"}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_data_sources_tenant ON data_sources(tenant_id);

CREATE TABLE plans (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    data_source_id  UUID NOT NULL REFERENCES data_sources(id),
    external_id     TEXT NOT NULL,  -- ID in the billing provider
    name            TEXT NOT NULL,
    billing_interval TEXT NOT NULL CHECK (billing_interval IN (
        'monthly', 'quarterly', 'semi_annual', 'annual', 'custom'
    )),
    billing_interval_months SMALLINT NOT NULL DEFAULT 1,
    amount_cents    BIGINT NOT NULL,
    currency        CHAR(3) NOT NULL,  -- ISO 4217
    is_trial        BOOLEAN NOT NULL DEFAULT false,
    trial_days      INT,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    metadata        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, data_source_id, external_id)
);

CREATE INDEX idx_plans_tenant ON plans(tenant_id);
```

## Customers & Subscriptions

```sql
CREATE TABLE customers (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    data_source_id  UUID NOT NULL REFERENCES data_sources(id),
    external_id     TEXT NOT NULL,  -- ID in the billing provider
    name            TEXT,
    email           TEXT,
    company         TEXT,
    country         CHAR(2),  -- ISO 3166-1 alpha-2
    city            TEXT,
    acquisition_channel TEXT,
    first_seen_at   TIMESTAMPTZ NOT NULL,
    status          TEXT NOT NULL DEFAULT 'active' CHECK (status IN (
        'lead', 'trialing', 'active', 'past_due', 'cancelled', 'churned', 'reactivated'
    )),
    churned_at      TIMESTAMPTZ,
    churn_type      TEXT CHECK (churn_type IN ('voluntary', 'involuntary')),
    churn_reason    TEXT,
    custom_attributes JSONB NOT NULL DEFAULT '{}',
    tags            TEXT[] NOT NULL DEFAULT '{}',
    is_deleted      BOOLEAN NOT NULL DEFAULT false,  -- GDPR soft delete
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, data_source_id, external_id)
);

CREATE INDEX idx_customers_tenant ON customers(tenant_id);
CREATE INDEX idx_customers_status ON customers(tenant_id, status);
CREATE INDEX idx_customers_country ON customers(tenant_id, country);
CREATE INDEX idx_customers_tags ON customers USING GIN (tags);
CREATE INDEX idx_customers_custom_attrs ON customers USING GIN (custom_attributes);

CREATE TABLE subscriptions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    customer_id     UUID NOT NULL REFERENCES customers(id),
    plan_id         UUID NOT NULL REFERENCES plans(id),
    data_source_id  UUID NOT NULL REFERENCES data_sources(id),
    external_id     TEXT NOT NULL,
    status          TEXT NOT NULL CHECK (status IN (
        'trialing', 'active', 'past_due', 'paused', 'cancelled', 'expired'
    )),
    quantity        INT NOT NULL DEFAULT 1,
    mrr_cents       BIGINT NOT NULL,  -- normalised to monthly
    currency        CHAR(3) NOT NULL,
    start_date      DATE NOT NULL,
    trial_end_date  DATE,
    current_period_start DATE NOT NULL,
    current_period_end   DATE NOT NULL,
    cancelled_at    TIMESTAMPTZ,
    cancellation_reason TEXT,
    cancel_at_period_end BOOLEAN NOT NULL DEFAULT false,
    paused_at       TIMESTAMPTZ,
    resume_at       TIMESTAMPTZ,
    discount_percent NUMERIC(5,2),
    discount_amount_cents BIGINT,
    metadata        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, data_source_id, external_id)
);

CREATE INDEX idx_subscriptions_tenant ON subscriptions(tenant_id);
CREATE INDEX idx_subscriptions_customer ON subscriptions(customer_id);
CREATE INDEX idx_subscriptions_plan ON subscriptions(plan_id);
CREATE INDEX idx_subscriptions_status ON subscriptions(tenant_id, status);
```

## Billing Events & MRR Ledger

```sql
CREATE TABLE billing_events (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    data_source_id  UUID NOT NULL REFERENCES data_sources(id),
    customer_id     UUID REFERENCES customers(id),
    subscription_id UUID REFERENCES subscriptions(id),

    -- CloudEvents envelope
    ce_source       TEXT NOT NULL,       -- e.g. 'stripe/acct_xxx'
    ce_type         TEXT NOT NULL,       -- e.g. 'customer.subscription.updated'
    ce_time         TIMESTAMPTZ NOT NULL,
    ce_specversion  TEXT NOT NULL DEFAULT '1.0',

    -- Normalised event classification
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

CREATE TABLE mrr_movements (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    customer_id     UUID NOT NULL REFERENCES customers(id),
    subscription_id UUID NOT NULL REFERENCES subscriptions(id),
    billing_event_id UUID REFERENCES billing_events(id),
    movement_date   DATE NOT NULL,
    movement_type   TEXT NOT NULL CHECK (movement_type IN (
        'new', 'expansion', 'contraction', 'churn', 'reactivation'
    )),
    amount_cents    BIGINT NOT NULL,  -- positive for new/expansion/reactivation, negative for contraction/churn
    currency        CHAR(3) NOT NULL,
    amount_base_cents BIGINT NOT NULL,  -- converted to tenant base currency
    fx_rate         NUMERIC(18,8) NOT NULL DEFAULT 1.0,
    plan_id         UUID REFERENCES plans(id),
    previous_plan_id UUID REFERENCES plans(id),
    previous_mrr_cents BIGINT,
    new_mrr_cents   BIGINT NOT NULL,
    churn_type      TEXT CHECK (churn_type IN ('voluntary', 'involuntary')),
    dunning_stage   TEXT,  -- for involuntary churn tracking
    segment_tags    TEXT[] NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (movement_date);

CREATE INDEX idx_mrr_movements_tenant_date ON mrr_movements(tenant_id, movement_date);
CREATE INDEX idx_mrr_movements_customer ON mrr_movements(customer_id);
CREATE INDEX idx_mrr_movements_type ON mrr_movements(tenant_id, movement_type);
CREATE INDEX idx_mrr_movements_plan ON mrr_movements(plan_id);
CREATE INDEX idx_mrr_movements_churn_type ON mrr_movements(tenant_id, churn_type) WHERE churn_type IS NOT NULL;
CREATE INDEX idx_mrr_movements_segment ON mrr_movements USING GIN (segment_tags);
```

## FX Rates & Dunning

```sql
CREATE TABLE fx_rates (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    from_currency   CHAR(3) NOT NULL,  -- ISO 4217
    to_currency     CHAR(3) NOT NULL,
    rate            NUMERIC(18,8) NOT NULL,
    rate_date       DATE NOT NULL,
    source          TEXT NOT NULL CHECK (source IN ('ecb', 'openexchangerates', 'manual', 'billing_provider')),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (from_currency, to_currency, rate_date, source)
);

CREATE INDEX idx_fx_rates_lookup ON fx_rates(from_currency, to_currency, rate_date);

CREATE TABLE dunning_attempts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    customer_id     UUID NOT NULL REFERENCES customers(id),
    subscription_id UUID NOT NULL REFERENCES subscriptions(id),
    billing_event_id UUID REFERENCES billing_events(id),
    dunning_sequence INT NOT NULL,  -- 1, 2, 3...
    step_type       TEXT NOT NULL CHECK (step_type IN (
        'payment_retry', 'email', 'sms', 'in_app', 'discount_offer', 'pause_offer', 'final_notice'
    )),
    attempted_at    TIMESTAMPTZ NOT NULL,
    outcome         TEXT NOT NULL CHECK (outcome IN ('pending', 'recovered', 'failed', 'skipped')),
    recovered_at    TIMESTAMPTZ,
    recovered_amount_cents BIGINT,
    offer_type      TEXT,
    offer_details   JSONB,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_dunning_tenant ON dunning_attempts(tenant_id);
CREATE INDEX idx_dunning_customer ON dunning_attempts(customer_id);
CREATE INDEX idx_dunning_subscription ON dunning_attempts(subscription_id);
CREATE INDEX idx_dunning_outcome ON dunning_attempts(tenant_id, outcome);
```

## Cohorts & Benchmarks

```sql
CREATE TABLE cohorts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    cohort_month    DATE NOT NULL,  -- first day of month
    plan_id         UUID REFERENCES plans(id),  -- NULL = all plans
    segment_key     TEXT,  -- custom segmentation dimension
    segment_value   TEXT,
    initial_customers INT NOT NULL,
    initial_mrr_cents BIGINT NOT NULL,
    currency        CHAR(3) NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, cohort_month, plan_id, segment_key, segment_value)
);

CREATE TABLE cohort_periods (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    cohort_id       UUID NOT NULL REFERENCES cohorts(id),
    period_number   INT NOT NULL,  -- months since cohort start (0, 1, 2...)
    period_month    DATE NOT NULL,
    active_customers INT NOT NULL,
    mrr_cents       BIGINT NOT NULL,
    expansion_cents BIGINT NOT NULL DEFAULT 0,
    contraction_cents BIGINT NOT NULL DEFAULT 0,
    churn_cents     BIGINT NOT NULL DEFAULT 0,
    reactivation_cents BIGINT NOT NULL DEFAULT 0,
    gross_retention_pct NUMERIC(7,4),
    net_retention_pct   NUMERIC(7,4),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (cohort_id, period_number)
);

CREATE INDEX idx_cohort_periods_cohort ON cohort_periods(cohort_id);

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
    metric_name     TEXT NOT NULL CHECK (metric_name IN (
        'mrr_growth_rate', 'gross_churn_rate', 'net_churn_rate', 'logo_churn_rate',
        'net_revenue_retention', 'gross_revenue_retention', 'ltv_months',
        'arpu_cents', 'trial_conversion_rate', 'expansion_rate',
        'quick_ratio', 'cac_payback_months'
    )),
    p10             NUMERIC(12,4),
    p25             NUMERIC(12,4),
    p50             NUMERIC(12,4),  -- median
    p75             NUMERIC(12,4),
    p90             NUMERIC(12,4),
    sample_size     INT NOT NULL,
    is_published    BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (period_month, revenue_band, vertical, metric_name)
);

CREATE INDEX idx_benchmarks_lookup ON benchmarks(revenue_band, vertical, metric_name);
```

## Product Usage & Forecasts

```sql
CREATE TABLE usage_events (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    customer_id     UUID NOT NULL REFERENCES customers(id),
    event_name      TEXT NOT NULL,  -- e.g. 'feature_used', 'login', 'api_call'
    event_date      DATE NOT NULL,
    event_count     INT NOT NULL DEFAULT 1,
    properties      JSONB NOT NULL DEFAULT '{}',
    source          TEXT NOT NULL CHECK (source IN ('segment', 'rudderstack', 'custom_api', 'csv_import')),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (event_date);

CREATE INDEX idx_usage_events_tenant_date ON usage_events(tenant_id, event_date);
CREATE INDEX idx_usage_events_customer ON usage_events(customer_id);
CREATE INDEX idx_usage_events_name ON usage_events(tenant_id, event_name);

CREATE TABLE forecasts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    created_by      UUID REFERENCES users(id),
    name            TEXT NOT NULL,
    scenario        TEXT NOT NULL CHECK (scenario IN ('base', 'optimistic', 'pessimistic', 'custom')),
    forecast_months INT NOT NULL DEFAULT 12,
    start_month     DATE NOT NULL,
    assumptions     JSONB NOT NULL,
    -- assumptions example: {
    --   "monthly_new_mrr_cents": 50000,
    --   "new_mrr_growth_rate": 0.05,
    --   "gross_churn_rate": 0.03,
    --   "expansion_rate": 0.02,
    --   "confidence_level": 0.90
    -- }
    results         JSONB NOT NULL,
    -- results example: [
    --   {"month": "2026-07-01", "mrr_cents": 1500000, "lower_bound": 1350000, "upper_bound": 1650000},
    --   ...
    -- ]
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_forecasts_tenant ON forecasts(tenant_id);
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
| Entity Management | 2 | tenants, users |
| Billing Sources & Plans | 2 | data_sources, plans |
| Customers & Subscriptions | 2 | customers, subscriptions |
| Billing Events & MRR | 2 | billing_events (partitioned), mrr_movements (partitioned) |
| FX & Dunning | 2 | fx_rates, dunning_attempts |
| Cohorts & Benchmarks | 3 | cohorts, cohort_periods, benchmarks |
| Product Usage & Forecasts | 2 | usage_events (partitioned), forecasts |
| AI & Audit | 2 | ai_suggestions, audit_log (partitioned) |
| **Total** | **17** | 4 partitioned tables |

---

## Key Design Decisions

1. **MRR movements as explicit rows** — each MRR change is a discrete record linking billing event → customer → subscription → plan, enabling five-movement waterfall queries with simple GROUP BY on movement_type and movement_date.

2. **Voluntary vs. involuntary churn on mrr_movements** — churn_type is stored directly on the movement row, not derived at query time, because the classification depends on dunning context available only at event processing time.

3. **Dunning attempts as separate table** — enables per-step analytics ("retry 2 recovers 18% of failed payments") independent of the billing event stream, supporting the key differentiator of dunning-stage recovery tracking.

4. **Historical FX rates preserved** — fx_rates stores daily rates by source, and mrr_movements records the fx_rate used at conversion time, so MRR can be recalculated with different rate sources without losing the original conversion.

5. **Cohort periods pre-computed** — cohort_periods materialises the retention grid so the cohort heatmap doesn't require scanning all mrr_movements on every dashboard load.

6. **Billing events use CloudEvents envelope** — ce_source/ce_type/ce_time fields enable vendor-neutral event ingestion while raw_payload preserves the original billing provider payload for debugging.

7. **Benchmarks as statistical distributions** — storing p10/p25/p50/p75/p90 per metric per revenue band per vertical enables peer comparison without exposing individual company data.

8. **Usage events partitioned by date** — product-usage data can be high-volume; date partitioning keeps queries fast and enables retention policies.

9. **Customer soft delete for GDPR** — is_deleted flag allows anonymising PII while preserving aggregate MRR and cohort metrics integrity.

10. **Plans normalised per billing source** — a customer on Stripe and another on Chargebee may have plans with the same name but different structures; normalising by data_source_id avoids conflation.
