# Subscription Analytics Platform — Phased Development Plan

> Project: 391-subscription-analytics-platform · Created: 2026-05-30
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

This plan synthesises `research.md`, `features.md`, `standards.md`, `README.md`, and the three data-model suggestions. It adopts **Data Model Suggestion 1 (Entity-Centric Normalized Relational)** as the canonical schema, because finance teams require auditable MRR calculations with full referential integrity and a clear audit trail from billing event → MRR movement → metric. The append-only `billing_events` table captures the audit-first benefit of the event-sourced alternative (Suggestion 3) without forcing every read through an event-replay projection.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Primary language | Python 3.12 | The core value (cohort survival analysis, LTV projection, churn classification, forecasting, anomaly detection) is statistical and ML-adjacent. `pandas`, `numpy`, `statsmodels`, and `lifetimes` — all named in research.md — are Python-first. The billing-source SDKs (Stripe, Chargebee, Recurly) all ship mature Python clients. |
| API framework | FastAPI | Async-native (needed for webhook ingestion and concurrent provider polling), generates an OpenAPI 3.1 document automatically (required by standards.md), and validates request/response bodies with Pydantic v2 using JSON Schema 2020-12 — the exact dialect standards.md mandates. |
| Data validation / schemas | Pydantic v2 | Single source of truth for both API I/O and the CloudEvents envelope. Emits JSON Schema 2020-12 for the public metric definitions standards.md calls for. |
| Primary database | PostgreSQL 16 | Data Model 1 relies on `UUID`, `JSONB`, `GIN` indexes, `CHECK` constraints, range partitioning (`billing_events`, `mrr_movements`, `usage_events`, `audit_log`), and `NUMERIC(18,8)` FX precision — all native to Postgres. SQLite cannot express partitioning or `JSONB` GIN indexing. |
| Migrations | Alembic | Standard for SQLAlchemy; supports the partitioned-table DDL and incremental schema evolution across phases. |
| ORM / query layer | SQLAlchemy 2.0 (Core + typed ORM) | Typed models map cleanly to the normalised schema. Core is used for the heavy analytical aggregation queries (waterfall, cohort grid) where raw SQL control matters. |
| Task queue | Celery + Redis | Webhook ingestion must return `200` fast and process asynchronously; provider backfills, nightly MRR reconciliation, cohort materialisation, FX refresh, and AI scoring are long-running. Celery beat handles the scheduled reconciliation jobs research.md assigns to Airflow/Prefect — adequate at MVP scale without a separate orchestrator. |
| Cache / broker | Redis 7 | Doubles as Celery broker and dashboard query cache (metric tiles, benchmark lookups). |
| LLM integration | Provider-agnostic client wrapping the Anthropic + OpenAI SDKs behind one interface | AI features (NL commentary, churn scoring rationale, anomaly explanation) must not hard-bind to one vendor. Prompt templates are versioned in code; `ai_suggestions.model_version` records which produced each output. |
| Frontend | Next.js 15 (React, TypeScript) + Recharts | Dashboard-heavy product: MRR waterfall, cohort heatmap grid, customer timeline drill-down, metric tiles. Server components render the shell; Recharts draws the waterfall/heatmap. Embeddable dashboards (v1.1) reuse the same chart components inside a sandboxed iframe + signed-JWT embed token. |
| AuthN / AuthZ | OAuth 2.0 Authorization Code + PKCE (RFC 9700) for user login via OIDC; API keys (RFC 7617 Basic) for programmatic access; HMAC-SHA256 for inbound webhook verification | Matches standards.md exactly: ChartMogul/Baremetrics use API-key Basic auth; Stripe webhooks use HMAC-SHA256; enterprise SSO needs OIDC. |
| Secrets | HashiCorp Vault (pluggable; env-var fallback in dev) | `data_sources.credentials_ref` / `webhook_secret_ref` store Vault references, never raw credentials (PCI DSS v4.0 requirement). |
| Containerisation | Docker + docker-compose | Self-hosted-first deployment. Compose wires api, worker, beat, postgres, redis, web. |
| Testing | pytest + pytest-asyncio + testcontainers; Playwright for web E2E | testcontainers spins a real Postgres for integration tests (partitioning/GIN behaviour cannot be mocked faithfully). Playwright covers dashboard flows. |
| Code quality | ruff (lint + format), mypy (strict), pre-commit | Standard modern Python toolchain. |
| Package / build | uv (Python), pnpm (web) | Fast, lockfile-based, reproducible. |
| Key libraries | `stripe`, `chargebee`, `recurly`, `pandas`, `numpy`, `statsmodels`, `lifetimes`, `python-jose` (JWT), `httpx`, `cloudevents` | Domain-specific per research.md and standards.md. |

### Project Structure

```
subscription-analytics-platform/
├── pyproject.toml
├── uv.lock
├── Dockerfile
├── docker-compose.yml
├── alembic.ini
├── README.md
├── openapi/
│   └── metric-definitions.schema.json   # published JSON Schema 2020-12 metric defs
├── migrations/                          # Alembic versions
│   └── versions/
├── src/
│   └── sap/
│       ├── __init__.py
│       ├── main.py                      # FastAPI app factory
│       ├── config.py                    # Pydantic Settings
│       ├── db/
│       │   ├── base.py                  # engine, session
│       │   ├── models/                  # SQLAlchemy ORM (one module per entity group)
│       │   │   ├── tenant.py
│       │   │   ├── data_source.py
│       │   │   ├── customer.py
│       │   │   ├── subscription.py
│       │   │   ├── billing_event.py
│       │   │   ├── mrr_movement.py
│       │   │   ├── dunning.py
│       │   │   ├── fx_rate.py
│       │   │   ├── cohort.py
│       │   │   ├── benchmark.py
│       │   │   ├── usage_event.py
│       │   │   ├── forecast.py
│       │   │   └── ai_audit.py
│       │   └── partitions.py            # partition management helpers
│       ├── schemas/                     # Pydantic request/response + CloudEvents
│       │   ├── cloudevents.py
│       │   └── ...
│       ├── auth/                        # OAuth/OIDC, API keys, embed JWT, RBAC
│       ├── connectors/                  # billing-source adapters
│       │   ├── base.py                  # Connector ABC + normaliser interface
│       │   ├── stripe.py
│       │   ├── chargebee.py
│       │   ├── recurly.py
│       │   ├── csv_import.py
│       │   └── registry.py
│       ├── ingestion/                   # webhook handlers, backfill, idempotency
│       ├── ledger/                      # MRR movement classifier + reconciliation
│       │   ├── classifier.py            # five-movement waterfall engine
│       │   ├── fx.py                    # currency normalisation
│       │   └── reconcile.py
│       ├── analytics/                   # metrics, cohorts, churn, benchmarks
│       │   ├── metrics.py               # MRR/ARR/LTV/ARPU/NRR
│       │   ├── cohorts.py
│       │   ├── churn.py                 # voluntary/involuntary + dunning
│       │   ├── benchmarks.py
│       │   └── forecast.py
│       ├── ai/                          # LLM client, prompts, scorers
│       │   ├── client.py
│       │   ├── prompts.py
│       │   ├── commentary.py
│       │   ├── churn_score.py
│       │   └── anomaly.py
│       ├── alerts/                      # anomaly + MRR movement alerting
│       ├── api/
│       │   └── v1/                      # routers per resource
│       ├── embed/                       # embeddable dashboard + SDK token issuance
│       ├── workers/                     # Celery tasks + beat schedule
│       └── gdpr/                        # erasure / anonymisation workflows
├── web/                                 # Next.js dashboard
│   ├── package.json
│   └── src/
│       ├── app/
│       ├── components/charts/           # Waterfall, CohortHeatmap, MetricTile, Timeline
│       └── lib/api-client.ts
└── tests/
    ├── unit/
    ├── integration/
    ├── e2e/
    └── fixtures/                        # sample Stripe/Chargebee/Recurly payloads
```

---

## Phase 1: Foundation, Schema & Multi-Tenancy

### Purpose
Establish the project skeleton, the canonical PostgreSQL schema from Data Model 1, configuration, and the multi-tenant data isolation that every later phase depends on. After this phase the database can be migrated up/down cleanly, a tenant and user can be created, and all later code has typed ORM models to build on.

### Tasks

#### 1.1 — Project scaffold, config, and Docker
**What**: Create the Python package, dependency manifest, Docker images, and compose stack.

**Design**:
- `pyproject.toml` declares dependencies (FastAPI, SQLAlchemy 2.0, Alembic, Pydantic v2, Celery, redis, httpx, pytest, ruff, mypy).
- `src/sap/config.py` — Pydantic `Settings`:
```python
class Settings(BaseSettings):
    database_url: PostgresDsn
    redis_url: RedisUrl
    secret_backend: Literal["vault", "env"] = "env"
    vault_addr: str | None = None
    jwt_signing_key: SecretStr
    default_base_currency: str = "USD"
    log_level: str = "INFO"
    model_config = SettingsConfigDict(env_prefix="SAP_", env_file=".env")
```
- `docker-compose.yml` services: `api` (uvicorn), `worker` (celery), `beat` (celery beat), `postgres:16`, `redis:7`, `web`.
- `src/sap/main.py` exposes `create_app()` with a `/healthz` route returning `{"status": "ok", "db": <bool>, "redis": <bool>}`.

**Testing**:
- `Unit: Settings loads from env vars with SAP_ prefix → correct typed values`
- `Unit: missing SAP_DATABASE_URL → ValidationError naming the field`
- `Integration: GET /healthz with db+redis up → 200 {"status":"ok","db":true,"redis":true}`
- `Integration: GET /healthz with redis down → 200 db:true, redis:false`
- `E2E: docker compose up → all services healthy, /healthz reachable`

#### 1.2 — Database schema & migrations
**What**: Implement all 17 tables from Data Model 1 as SQLAlchemy ORM models plus the initial Alembic migration, including the 4 range-partitioned tables.

**Design**:
- One ORM module per entity group (see structure). Use the exact column types, `CHECK` constraints, uniqueness, and indexes from `data-model-suggestion-1.md`.
- Partitioned tables (`billing_events`, `mrr_movements`, `usage_events`, `audit_log`) declared with `postgresql_partition_by`. `db/partitions.py` provides `ensure_monthly_partition(table, month)` creating `<table>_YYYY_MM` partitions on demand (called from ingestion + a beat job).
- Enum-like `CHECK` values mirrored as Python `StrEnum`s for type safety (`MovementType`, `EventCategory`, `ChurnType`, `DunningStepType`, `Role`, `Provider`, etc.).
- Money is always `BIGINT` cents; FX rates `NUMERIC(18,8)`.

**Testing**:
- `Integration (real PG via testcontainers): alembic upgrade head → all 17 tables + indexes exist`
- `Integration: alembic downgrade base then upgrade head → idempotent, no errors`
- `Integration: insert mrr_movement with movement_type='invalid' → CHECK violation`
- `Integration: insert billing_event for 2026-07 → ensure_monthly_partition creates billing_events_2026_07, row lands in it`
- `Unit: MovementType StrEnum values exactly match the CHECK list`

#### 1.3 — Tenant context & row-level isolation
**What**: A request/worker-scoped tenant context that scopes every query to one `tenant_id`.

**Design**:
- `TenantContext` contextvar set from the authenticated principal (Phase 2) or task header.
- A SQLAlchemy session factory that injects `tenant_id` filters via a query helper `tenant_scoped(session, Model)`; raises `TenantContextMissing` if unset on a tenant-owned table.
- `tenants` and `users` CRUD service functions; `create_tenant(name, slug, base_currency, plan)` validates `base_currency` against ISO 4217.

**Testing**:
- `Unit: tenant_scoped query without context → TenantContextMissing`
- `Integration: two tenants, customers inserted for each → tenant A query returns only A's rows`
- `Unit: create_tenant with base_currency='XX' → ValidationError (not ISO 4217)`
- `Unit: create_tenant duplicate slug → IntegrityError surfaced as 409-mapped domain error`

---

## Phase 2: Authentication, Authorisation & Audit

### Purpose
Secure the platform before any billing data flows in. Implements OIDC login, API keys, RBAC across the six roles, the embed-token issuer scaffold, and the append-only audit log. After this phase every endpoint can require a principal and role, and every state change is recorded.

### Tasks

#### 2.1 — User auth (OIDC + session) and API keys
**What**: OAuth 2.0 Authorization Code + PKCE login via an OIDC provider, plus issuable API keys.

**Design**:
- `auth/oidc.py`: `/auth/login` → redirect with PKCE challenge; `/auth/callback` → exchange code, verify ID token, upsert `users` row, issue a platform session JWT (HS256 via `jwt_signing_key`, 1h expiry, refresh cookie).
- API keys: `POST /v1/api-keys` creates a key `sap_live_<random>`, stores only a SHA-256 hash + prefix; presented via RFC 7617 Basic (key as username).
- `auth/principal.py`: `get_principal()` FastAPI dependency resolving either a session JWT or an API key into `Principal(tenant_id, user_id, role, auth_method)`.

**Testing**:
- `Unit: PKCE verifier/challenge pair validates; tampered challenge → rejected`
- `Integration (mocked OIDC): valid callback → user upserted, session cookie set, 302`
- `Integration: API key Basic auth with valid key → principal resolved`
- `Integration: API key with wrong hash → 401`
- `Unit: stored key value is a hash, never the raw key`

#### 2.2 — RBAC
**What**: Role-based authorisation for the six roles (`owner, admin, analyst, viewer, api_service, embed_viewer`).

**Design**:
- Permission matrix `PERMISSIONS: dict[Role, set[Permission]]`; `require(*perms)` dependency factory returning 403 on failure.
- `embed_viewer` is read-only and scoped to a single dashboard via embed-token claims (Phase 9).

**Testing**:
- `Unit: viewer lacks manage:data_source → require() raises 403`
- `Unit: owner has all permissions`
- `Integration: analyst POST /v1/data-sources → 403; admin → 201`

#### 2.3 — Audit log
**What**: Append-only `audit_log` writes for every mutating action.

**Design**:
- `audit.record(actor_type, actor_id, action, resource_type, resource_id, changes, request)` writes one partitioned row; `changes` is a JSON diff.
- FastAPI middleware captures IP/user-agent; mutating service methods call `audit.record`.

**Testing**:
- `Integration: create data_source → audit_log row actor_type='user', action='data_source.create'`
- `Integration: audit row for 2026-07 lands in audit_log_2026_07 partition`
- `Unit: audit changes diff omits credential fields (no secrets logged)`

---

## Phase 3: Billing Connectors & Event Ingestion

### Purpose
The platform's lifeblood: ingest billing events from Stripe, Chargebee, and Recurly (the three MVP connectors from features.md) and normalise them into the CloudEvents-enveloped `billing_events` table with idempotency. After this phase raw events from any of the three providers, plus CSV import, land in a unified, replayable store.

### Tasks

#### 3.1 — Connector abstraction & CloudEvents envelope
**What**: Define the connector interface and the normalised event model.

**Design**:
```python
class NormalisedEvent(BaseModel):
    ce_source: str            # "stripe/acct_xxx"
    ce_type: str              # provider-native type
    ce_time: datetime
    ce_specversion: str = "1.0"
    event_category: EventCategory   # mapped to the 25 CHECK categories
    external_customer_id: str | None
    external_subscription_id: str | None
    idempotency_key: str
    raw_payload: dict

class Connector(ABC):
    provider: Provider
    @abstractmethod
    def verify_webhook(self, body: bytes, headers: Mapping) -> bool: ...   # HMAC-SHA256
    @abstractmethod
    def parse_webhook(self, body: bytes) -> list[NormalisedEvent]: ...
    @abstractmethod
    async def backfill(self, since: datetime, cursor: str | None) -> AsyncIterator[tuple[list[NormalisedEvent], str]]: ...
    @abstractmethod
    def map_category(self, native_type: str) -> EventCategory: ...
```
- `registry.py` maps `Provider → Connector` class.

**Testing**:
- `Unit: map_category('customer.subscription.deleted') → 'subscription_cancelled' (Stripe)`
- `Unit: unknown native type → 'subscription_updated' fallback + warning logged`
- `Fixture: parse committed Stripe payload → NormalisedEvent with correct ce_* fields`

#### 3.2 — Stripe, Chargebee, Recurly connectors
**What**: Concrete adapters implementing webhook verification, parsing, category mapping, and backfill.

**Design**:
- Stripe: HMAC-SHA256 over `Stripe-Signature` with tolerance window; backfill via `Subscription.list`/`Event.list` cursors.
- Chargebee: HTTP Basic, webhook by shared secret; backfill via `subscription.list` offsets.
- Recurly: API key, signed webhooks; backfill via `list_subscriptions` cursors.
- Each maps native event taxonomies onto the 25 `event_category` values. Category-map tables committed as fixtures and unit-tested.

**Testing**:
- `Unit (per provider): valid signature → verify_webhook true; tampered body → false`
- `Fixture: each provider's invoice.payment_failed sample → event_category='payment_failed'`
- `Integration (mocked HTTP): backfill yields two cursor pages → all events normalised, cursor persisted to data_sources.sync_cursor`

#### 3.3 — Webhook endpoint, idempotency & async dispatch
**What**: `POST /v1/webhooks/{data_source_id}` that verifies, persists, and enqueues processing.

**Design**:
- Resolve `data_source` → connector; `verify_webhook` else `401`. Persist `NormalisedEvent`s into `billing_events` with `UNIQUE(tenant_id, data_source_id, idempotency_key)` (duplicates ignored). Enqueue `process_event` Celery task. Return `200` within the verify+insert path; classification is async.
- Upsert referenced `customers`/`subscriptions`/`plans` (external_id resolution) inside the processing task.

**Testing**:
- `Integration (mocked verify): valid webhook → 200, billing_event row inserted, process_event enqueued`
- `Integration: invalid signature → 401, no row, no enqueue`
- `Integration: same idempotency_key twice → one row, 200 both times`
- `Integration: webhook for unknown data_source_id → 404`

#### 3.4 — CSV import & backfill orchestration
**What**: CSV upload connector and a backfill task for first-time source connection.

**Design**:
- `POST /v1/data-sources/{id}/import` accepts a CSV mapped to a documented column spec → `NormalisedEvent`s.
- `backfill_source(data_source_id)` task drives the connector's `backfill`, updates `status` pending→syncing→active and `last_sync_at`.

**Testing**:
- `Fixture: well-formed CSV → N events ingested`
- `Unit: CSV missing required column → 422 listing the column`
- `Integration: backfill task → data_source.status transitions to 'active', sync_cursor set`

---

## Phase 4: MRR Ledger & Movement Classifier

### Purpose
The analytical heart. Transform the normalised event stream into the five-movement MRR ledger (new, expansion, contraction, churn, reactivation) with multi-currency normalisation. After this phase every billing event produces auditable `mrr_movements` rows tied back to their source event.

### Tasks

#### 4.1 — FX normalisation
**What**: Convert plan/subscription amounts into the tenant base currency using historical rates.

**Design**:
- `fx.convert(amount_cents, from_ccy, to_ccy, on_date) -> (amount_base_cents, rate)`; looks up `fx_rates` for the nearest date ≤ `on_date`, falls back to most recent, raises `MissingFXRate` if none.
- `refresh_fx_rates()` beat task pulls daily ECB rates into `fx_rates(source='ecb')`.

**Testing**:
- `Unit: convert 1000 EUR→USD at known rate → correct base cents + rate recorded`
- `Unit: same-currency convert → rate=1.0, unchanged`
- `Unit: no rate available → MissingFXRate`
- `Integration (mocked ECB): refresh_fx_rates → rows upserted, UNIQUE respected`

#### 4.2 — Five-movement classifier
**What**: Classify each processed event into zero or more `mrr_movements`.

**Design**: Deterministic rules driven by the subscription's prior vs. new MRR:
| Transition | Movement |
|-----------|----------|
| no active sub → active sub | `new` |
| active, MRR↑ (upgrade/qty↑) | `expansion` |
| active, MRR↓ (downgrade/qty↓/discount) | `contraction` |
| active → cancelled/expired | `churn` (sign negative) |
| previously churned → active again | `reactivation` |
- Each movement stores `previous_mrr_cents`, `new_mrr_cents`, `amount_cents` (delta, signed), `amount_base_cents`, `fx_rate`, `plan_id`, `previous_plan_id`, `segment_tags` (snapshot of customer tags/plan).
- Churn movements set `churn_type` and `dunning_stage` (filled by Phase 5 if involuntary; default `voluntary` for explicit cancellation).
- Idempotent: re-processing an event upserts on `billing_event_id`.

**Testing**:
- `Unit: trial→paid first charge → one 'new' movement = plan MRR`
- `Unit: $50→$100 plan change → 'expansion' amount=+5000, previous_mrr=5000, new_mrr=10000`
- `Unit: qty 3→1 → 'contraction' negative delta`
- `Unit: cancel active sub → 'churn' negative = -new_mrr`
- `Unit: churned customer resubscribes → 'reactivation' positive`
- `Unit: reprocess same event → no duplicate movement`
- `Integration: multi-currency sub → amount_base_cents converted, fx_rate stored`

#### 4.3 — Nightly reconciliation
**What**: A beat job that recomputes movements for unprocessed events and reconciles ledger MRR against subscription state.

**Design**:
- `reconcile_tenant(tenant_id, as_of)`: process all `is_processed=false` events in `ce_time` order; assert `sum(mrr_movements up to as_of) == sum(active subscriptions.mrr_cents)`; emit a `data_quality_issue` AI suggestion on mismatch.

**Testing**:
- `Integration: backlog of 100 events → all processed, is_processed=true`
- `Integration: injected discrepancy → reconciliation flags data_quality_issue with delta in evidence`

---

## Phase 5: Churn Classification & Dunning Analytics

### Purpose
Deliver the platform's primary differentiator from features.md: voluntary vs. involuntary churn classification and dunning-stage recovery analytics. After this phase the platform can answer "which recovery step recovers what share of involuntary churn?".

### Tasks

#### 5.1 — Dunning lifecycle tracking
**What**: Record dunning attempts from `dunning_*` events into `dunning_attempts`.

**Design**:
- Events `dunning_started/dunning_step/dunning_recovered/dunning_exhausted` and `payment_failed/payment_recovered` create/update `dunning_attempts(dunning_sequence, step_type, outcome, recovered_at, recovered_amount_cents)`.
- A `payment_failed` opens a dunning episode; `payment_recovered` closes it `recovered`; `dunning_exhausted` → involuntary churn.

**Testing**:
- `Integration: payment_failed → email → retry 2 → recovered → 3 attempts, final outcome 'recovered'`
- `Integration: payment_failed → exhausted → attempts outcome 'failed', churn flagged involuntary`

#### 5.2 — Voluntary/involuntary classification
**What**: Set `churn_type` on churn movements using dunning context.

**Design**:
- Explicit `subscription_cancelled` not preceded by an open dunning episode → `voluntary`.
- Churn following `dunning_exhausted` / unrecovered `payment_failed` → `involuntary`, with `dunning_stage` = last reached step.

**Testing**:
- `Unit: cancel with no open dunning → voluntary`
- `Unit: churn after exhausted dunning → involuntary, dunning_stage='final_notice'`

#### 5.3 — Dunning recovery analytics
**What**: Aggregate recovery effectiveness per step.

**Design**:
- `dunning_recovery_breakdown(tenant_id, period) -> [{step_type, attempts, recovered, recovery_rate, recovered_mrr_cents}]`.
- `GET /v1/analytics/dunning?from=&to=` returns the breakdown.

**Testing**:
- `Integration: seeded attempts → retry_2 recovery_rate computed correctly`
- `Integration: API returns ordered breakdown by dunning_sequence`

---

## Phase 6: Core Metrics, Cohorts & Public API

### Purpose
Expose the table-stakes metric suite (MRR, ARR, LTV, ARPU, churn rate, NRR), the MRR waterfall, and the cohort net-MRR-retention grid via a documented OpenAPI 3.1 REST surface. After this phase the platform is queryable end-to-end and ships a machine-readable metric-definitions schema — the reference-implementation opportunity standards.md identifies.

### Tasks

#### 6.1 — Metric engine
**What**: Compute the core SaaS metrics from the ledger.

**Design**:
```python
def mrr(tenant_id, on: date, segment: Segment | None) -> int           # base cents
def arr(tenant_id, on: date) -> int                                     # mrr * 12
def arpu(tenant_id, on: date) -> int
def ltv(tenant_id, on: date) -> int                                     # arpu / gross_churn_rate
def gross_churn_rate(tenant_id, period: Period) -> Decimal
def net_revenue_retention(tenant_id, period: Period) -> Decimal
def mrr_waterfall(tenant_id, period: Period, segment) -> Waterfall      # by movement_type
```
- Definitions published as JSON Schema 2020-12 in `openapi/metric-definitions.schema.json` (e.g. NRR = (start + expansion + reactivation − contraction − churn) / start).

**Testing**:
- `Unit: fixture ledger → mrr_waterfall sums match expected per movement_type`
- `Unit: NRR with expansion>churn → >100%`
- `Unit: gross_churn_rate zero starting MRR → handled (returns 0, no div-by-zero)`
- `Unit: segment filter restricts to matching segment_tags`

#### 6.2 — Cohort engine & materialisation
**What**: Compute and persist the cohort net-MRR-retention grid.

**Design**:
- `build_cohorts(tenant_id, dimension)` groups customers by signup month × plan (configurable cohort-start = first charge), writes `cohorts` + `cohort_periods` with `gross_retention_pct`/`net_retention_pct` per period.
- Materialised nightly; `GET /v1/analytics/cohorts` reads `cohort_periods`.

**Testing**:
- `Integration: 3 cohorts over 3 months → period 0 = 100%, later periods reflect expansion/churn`
- `Unit: net retention nets expansion+reactivation against contraction+churn`
- `Integration: cohort grid served from materialised table without scanning mrr_movements`

#### 6.3 — Customer timeline
**What**: Per-customer subscription history with churn-reason capture.

**Design**:
- `GET /v1/customers/{id}/timeline` returns ordered events: plan changes, pauses, cancellations, reactivations, dunning, churn_reason.

**Testing**:
- `Integration: customer with upgrade+pause+churn → timeline ordered by ce_time with churn_type+reason`

#### 6.4 — REST API & OpenAPI document
**What**: Versioned `/v1` routers; emit `openapi.json`.

**Design**:
- Routers: data-sources, customers, subscriptions, metrics, cohorts, dunning, exports. All responses Pydantic-typed → OpenAPI 3.1. CSV export endpoints for portability (features.md).

**Testing**:
- `Integration: GET /openapi.json → valid OAS 3.1, all /v1 paths present`
- `Integration: GET /v1/metrics?on=2026-05-01 → typed payload, 200`
- `Integration: CSV export → correct headers and row count`

---

## Phase 7: Dashboard Frontend

### Purpose
The finance-team-facing UI: metric tiles, MRR waterfall, cohort heatmap, customer drill-down. After this phase the product is usable by non-technical operators.

### Tasks

#### 7.1 — App shell, auth, API client
**What**: Next.js shell with OIDC login and a typed API client.

**Design**:
- Generate `web/src/lib/api-client.ts` from `openapi.json`. Auth via session cookie; role-gated nav.

**Testing**:
- `E2E (Playwright): login → dashboard renders for analyst role`
- `E2E: viewer cannot see data-source management nav`

#### 7.2 — Charts
**What**: Waterfall, cohort heatmap, metric tiles, customer timeline.

**Design**:
- `Waterfall` colour-codes the five movements; `CohortHeatmap` colours retention %; `MetricTile` shows value + period-over-period delta sparkline; `Timeline` renders the customer history.

**Testing**:
- `E2E: dashboard shows MRR tile matching API value`
- `E2E: waterfall segments match /v1/analytics/waterfall response`
- `E2E: cohort heatmap cell count = months × cohorts`
- `Component: Waterfall with zero movements → empty state, no crash`

---

## Phase 8: AI-Native Layer

### Purpose
Deliver the AI differentiators from features.md: natural-language MRR commentary, automated churn-risk scoring, and anomaly detection on daily MRR movements. After this phase the platform produces explainable, board-ready insight and proactive alerts.

### Tasks

#### 8.1 — Provider-agnostic LLM client & prompts
**What**: One interface over Anthropic/OpenAI with versioned prompt templates.

**Design**:
- `ai/client.py`: `complete(system, user, model) -> str`; provider chosen by config. `ai/prompts.py` holds versioned templates; every output records `model_version` on `ai_suggestions`.

**Testing**:
- `Unit (mocked): complete routes to configured provider`
- `Unit: prompt template renders with metric context, no unfilled placeholders`

#### 8.2 — NL MRR commentary
**What**: Generate board-ready commentary grounded in computed metrics.

**Design**:
- `generate_commentary(tenant_id, period)` feeds the waterfall + segment breakdown into a template; produces text like *"MRR declined 3.2% this month primarily due to annual-plan churn in the SMB segment"*; stored as `ai_suggestions(suggestion_type='mrr_commentary')` with `evidence` = the metric numbers used.

**Testing**:
- `Integration (mocked LLM): commentary references the actual top movement driver from evidence`
- `Unit: evidence JSON contains the figures cited`

#### 8.3 — Churn-risk scoring
**What**: Score active customers combining billing patterns, product usage, and dunning history.

**Design**:
- Feature vector per customer (recent contraction, failed payments, usage trend from `usage_events`, tenure); gradient-boosted or logistic baseline producing 0–1 risk; LLM generates the rationale. Stored as `ai_suggestions(suggestion_type='churn_risk', confidence=...)`.

**Testing**:
- `Unit: customer with declining usage + failed payment → higher score than healthy customer`
- `Integration: scoring run writes churn_risk suggestions with confidence`

#### 8.4 — Anomaly detection on daily MRR
**What**: Flag anomalous day-over-day MRR deviations (pipeline-issue early warning).

**Design**:
- Rolling z-score / seasonal-decomposition on daily net MRR; deviations beyond threshold → `ai_suggestions(suggestion_type='anomaly_detected', severity)`.

**Testing**:
- `Unit: injected 40% single-day drop → anomaly flagged with z-score in evidence`
- `Unit: normal variation → no anomaly`

---

## Phase 9: Advanced Analytics — Benchmarks, Forecasting, Multi-Currency, Embedding

### Purpose
Ship the v1.1 "should-have" and selected "nice-to-have" features: peer benchmarking, scenario forecasting, full multi-currency reporting, expansion-signal detection, and embeddable dashboards. After this phase the platform reaches feature parity-plus with incumbents.

### Tasks

#### 9.1 — Benchmarking
**What**: Anonymised peer comparison by revenue band × vertical.

**Design**:
- `compute_tenant_metrics_for_benchmark` contributes anonymised per-metric values into the shared `benchmarks` distribution (p10–p90); `GET /v1/analytics/benchmarks` returns the tenant's percentile vs. peers. Minimum `sample_size` gate before publishing (privacy).

**Testing**:
- `Unit: percentiles computed correctly from a known sample`
- `Integration: benchmark below sample_size threshold → not published / 404`
- `Unit: no individual tenant value is exposed in the response`

#### 9.2 — Scenario forecasting
**What**: MRR forecast with confidence intervals under base/optimistic/pessimistic scenarios.

**Design**:
- `forecast(tenant_id, assumptions, months)` projects MRR from new/growth/churn/expansion assumptions; CI from historical volatility; persisted to `forecasts.results` as month/mrr/lower/upper triples.

**Testing**:
- `Unit: zero churn + fixed new MRR → linear projection`
- `Unit: results length = forecast_months, bounds monotonic around mean`

#### 9.3 — Expansion-signal detection
**What**: Correlate product-usage milestones with upgrade probability.

**Design**:
- Join `usage_events` with subsequent expansion movements; surface `ai_suggestions(suggestion_type='expansion_signal')` for accounts hitting usage thresholds preceding historical upgrades.

**Testing**:
- `Integration: account exceeding usage threshold with historical upgrade correlation → expansion_signal suggestion`

#### 9.4 — Embeddable dashboards & SDK
**What**: Signed-token embedding of waterfall/cohort charts inside customers' products.

**Design**:
- `POST /v1/embed/tokens` issues a short-lived JWT scoped to `tenant_id` + dashboard + filters (role `embed_viewer`). `web/embed/[token]` renders a sandboxed read-only dashboard; a small JS SDK mounts it via iframe.

**Testing**:
- `Unit: embed token claims restrict to one dashboard + tenant`
- `Integration: expired/invalid embed token → 401`
- `E2E: embed page renders cohort chart read-only, no nav`

---

## Phase 10: Compliance, GDPR & Hardening

### Purpose
Meet the security/privacy obligations from standards.md (GDPR, CCPA, ISO 27701, PCI DSS) and production-harden the platform. After this phase the platform is enterprise-deployable.

### Tasks

#### 10.1 — GDPR erasure / anonymisation
**What**: Right-to-erasure that anonymises PII without corrupting aggregate metrics.

**Design**:
- `erase_customer(customer_id)` sets `is_deleted=true`, nulls/hashes PII (name/email/company), retains `mrr_movements`/cohort aggregates intact. Cascade documented; audit-logged. `DELETE /v1/customers/{id}` (admin).

**Testing**:
- `Integration: erase customer → PII nulled, MRR aggregates unchanged, cohort_periods stable`
- `Integration: erasure recorded in audit_log`

#### 10.2 — Secrets via Vault & PCI exclusion
**What**: Route all credentials through the secret backend; ensure no cardholder data is stored.

**Design**:
- `data_sources.credentials_ref`/`webhook_secret_ref` resolve through `secret_backend`; ingestion strips card/PAN fields from `raw_payload` (PCI DSS v4.0 — store only tokenised references).

**Testing**:
- `Unit: ingested payload with a card field → field dropped before persistence`
- `Integration (mocked Vault): connector loads credential by ref, raw secret never in DB`

#### 10.3 — Rate limiting, alerting delivery & observability
**What**: API rate limits, Slack/email alert delivery for anomalies and MRR-movement thresholds, structured logging/metrics.

**Design**:
- Redis token-bucket per API key. Alert dispatcher sends anomaly/MRR-movement-threshold `ai_suggestions` to Slack/email. JSON structured logs + Prometheus `/metrics`.

**Testing**:
- `Integration: exceed rate limit → 429 with Retry-After`
- `Integration (mocked Slack): critical anomaly → Slack message posted`
- `Integration: GET /metrics → Prometheus exposition format`

---

## Phase Summary & Dependencies

```
Phase 1: Foundation, Schema & Multi-Tenancy   ─── required by everything
    │
Phase 2: Auth, RBAC & Audit                   ─── requires Phase 1
    │
Phase 3: Connectors & Event Ingestion         ─── requires Phase 2
    │
Phase 4: MRR Ledger & Movement Classifier     ─── requires Phase 3
    │
Phase 5: Churn & Dunning Analytics            ─── requires Phase 4
    │
Phase 6: Core Metrics, Cohorts & Public API   ─── requires Phase 4 (uses Phase 5 for churn split)
    ├── Phase 7: Dashboard Frontend           ─── requires Phase 6
    └── Phase 8: AI-Native Layer              ─── requires Phase 6 (can parallel with Phase 7)
         │
Phase 9: Advanced Analytics (benchmarks,      ─── requires Phase 6 + Phase 8
         forecasting, multi-currency, embed)
    │
Phase 10: Compliance, GDPR & Hardening        ─── requires Phase 2 + Phase 3 (finalised last)
```

**Parallelism opportunities:**
- Phases 7 and 8 can be developed concurrently once Phase 6 is complete.
- Within Phase 3, the three connectors (3.2) can be built in parallel against the shared interface (3.1).
- Phase 10.1 (GDPR) and 10.2 (secrets/PCI) are independent and can proceed in parallel.

---

## Definition of Done (per phase)

1. All tasks in the phase are implemented.
2. All unit and integration tests pass (`pytest`); web phases pass Playwright E2E.
3. `ruff check` and `ruff format --check` pass.
4. `mypy --strict` passes for `src/sap`.
5. `docker compose build` succeeds; affected services start healthy.
6. The phase's headline capability works end-to-end against a real Postgres (testcontainers).
7. New configuration options are documented in `README.md` / `.env.example`.
8. New API endpoints appear in the generated `openapi.json` (OAS 3.1) with typed schemas.
9. Alembic migration(s) created and reversible (`upgrade`/`downgrade` verified).
10. Every new mutating action writes an `audit_log` entry; no secrets or cardholder data are persisted.
