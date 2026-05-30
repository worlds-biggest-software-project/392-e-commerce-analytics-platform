# E-Commerce Analytics Platform — Phased Development Plan

> Project: 392-e-commerce-analytics-platform · Created: 2026-05-30
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

This plan synthesises `research.md`, `features.md`, `standards.md`, `README.md`, and the three data-model suggestions. It adopts **Data Model Suggestion 1 (Entity-Centric Normalised Relational)** as the canonical schema because full SQL access to every dimension is a stated core differentiator over the proprietary incumbents (Triple Whale, Glew, Improvado). Where rapid config iteration is needed (connector config, attribution model parameters, AI evidence), the schema uses targeted `JSONB` columns as recommended by Suggestion 2. Suggestion 3's event-sourced ideas inform the immutable `behavioural_events` and `audit_log` partitioned tables but are not adopted wholesale.

---

## Core Requirements (synthesis)

1. **What it does** — Consolidates shopper-behaviour events, marketing-attribution signals, conversion-funnel metrics, and inventory-performance data into a single, SQL-queryable multi-tenant model so merchants can connect early browse sessions to delayed conversions, evaluate channel quality by repeat-purchase, and detect over-advertised-but-out-of-stock products.
2. **Who uses it** — DTC operators (primary), multi-channel retailers (owned store + Amazon + social), and enterprise retail marketing teams. Personas: store owner/operator, marketing analyst, data analyst (SQL/warehouse), and a read-only executive viewer.
3. **Key differentiators** — (a) demand-side + supply-side analytics in one model (vs. Triple Whale's missing inventory); (b) open, fully SQL-accessible data model (vs. proprietary); (c) AI-native natural-language analytics and inventory-to-ad-spend automation; (d) cookieless-first server-side capture.
4. **MVP feature set** — first-party/server-side event capture; Shopify/WooCommerce/BigCommerce connectors; multi-touch attribution (last-click, first-click, linear); conversion funnel analysis segmented by source/device/category; cohort retention + LTV by acquisition month; core metrics (GMV, AOV, conversion rate, CAC by channel).
5. **Post-MVP** — creative-level paid-media analytics; inventory performance + stockout alerting; customer segmentation with exportable lists; warehouse export (Snowflake/BigQuery); probabilistic attribution; then AI NL interface, real-time inventory-to-ad-spend pausing, SKU demand forecasting, cross-device identity, marketplace attribution.
6. **Deployment model** — Self-hosted, container-first (Docker Compose) SaaS-style multi-tenant application; optional managed cloud. PostgreSQL is the operational store; Snowflake/BigQuery are *export targets*, not required for the platform to run.
7. **Integration surface** — Inbound: Shopify (GraphQL Admin + webhooks), WooCommerce REST, BigCommerce API, first-party JS pixel, server-side HTTP event API (Segment-spec compatible), GA4 Measurement Protocol ingest, Meta/Google/TikTok Ads (spend + creative). Outbound: Snowflake/BigQuery export, Slack/email alerts, LLM providers, MCP server.
8. **Standards compliance** — Segment E-Commerce Spec v2 (event taxonomy), GA4 Measurement Protocol (ingest normalisation), OpenAPI 3.1/3.2 + JSON Schema 2020-12 (API surface), OAuth 2.0 (connectors), OIDC (user auth), ISO 4217 (currency), ISO 3166-1 (country), GDPR/CCPA (consent + erasure), PCI DSS v4.0 (never capture card data; operate upstream of payment capture), OWASP Top 10 (API hardening), MCP (AI tool surface).
9. **Data model** — Suggestion 1: 15 tables, 5 partitioned (`sessions`, `orders`, `touchpoints`, `behavioural_events`, `audit_log`).

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Primary language | Python 3.12 | Forecasting (scikit-learn, Prophet), attribution MMM (Google Meridian, Apache 2.0), and LLM tooling all live in Python. A single language across ingestion, analytics, and ML avoids a polyglot split. |
| API framework | FastAPI | Native OpenAPI 3.1 + JSON Schema 2020-12 generation (a stated standards requirement), async I/O for webhook/connector fan-out, Pydantic validation for event payloads. |
| Data validation | Pydantic v2 | Enforces Segment-spec event schemas and API contracts; serialises directly into OpenAPI. |
| Operational DB | PostgreSQL 16 | Suggestion 1 relies on PG features: declarative range partitioning, `JSONB` + GIN, `INET`, array columns, partial indexes, `gen_random_uuid()`. Zero-extra-ops; the SQL-first differentiator is delivered directly. |
| Migrations | Alembic | Versioned schema migrations; required given 15 tables + partition management. |
| ORM / query | SQLAlchemy 2.0 (core + ORM) | ORM for entity CRUD; Core/raw SQL for heavy analytical aggregations (funnels, cohorts) where the SQL-first design matters. |
| Task queue | Celery + Redis | Async connector syncs, attribution recomputation, forecast training, alert evaluation, and warehouse export are long-running; Celery beat drives scheduled syncs. |
| Real-time stream | Redis Streams (MVP) → Apache Kafka (scale) | Inventory-change and stockout-vs-spend alerting need low-latency fan-out. Redis Streams keeps MVP ops light; abstract behind a `StreamProducer` interface to swap in Kafka later (research §5). |
| Forecasting | scikit-learn + Prophet | SKU-level demand forecasting and reorder points (README AI-native section). |
| Attribution MMM | Google Meridian (Apache 2.0) | Legally clear channel-level media-mix modelling for probabilistic attribution; no patent encumbrance (features.md Legal & IP). |
| Cache | Redis | Dashboard query result cache + rate-limit buckets; shares the Celery broker instance. |
| LLM access | Provider-abstracted client (Anthropic default) | NL analytics + AI suggestions; abstraction allows swapping providers and self-hosted models. |
| AI tool surface | MCP server (Python SDK) | `query_sales_metrics`, `get_funnel_report`, `get_product_performance` per standards.md, exposing analytics to Claude/Cursor. |
| Frontend | Next.js 16 (App Router) + TypeScript + Tailwind + shadcn/ui | Merchant-facing dashboard SPA: metric tiles, attribution waterfall, funnel, cohort heatmap, inventory, NL chat. Recharts for charting. |
| Auth | OIDC (Authlib) + JWT sessions | Federated SSO for enterprise (Okta/Auth0/Google) per standards.md; JWT for API. Per-tenant RBAC: owner/admin/analyst/viewer/api_service. |
| Connector framework | Custom `BaseConnector` + Airbyte for long-tail | Native connectors for the 3 primary commerce platforms (control over webhooks/latency); Airbyte for long-tail ad networks/marketplaces (research §5). |
| Warehouse export | dbt-core models + warehouse writers | dbt builds funnel/cohort marts; export writers push to Snowflake/BigQuery (research §5, README). |
| Containerisation | Docker + Docker Compose | Self-hosted deployment is the stated model; Compose wires api, worker, beat, postgres, redis, frontend. |
| Testing | pytest + pytest-asyncio + testcontainers; Playwright (FE) | Unit + integration (real PG/Redis via testcontainers) + E2E browser flows. |
| Quality tools | ruff (lint+format), mypy (strict), pre-commit | Single fast toolchain for Python; ESLint + Prettier for the frontend. |
| Package manager | uv (Python), pnpm (frontend) | Fast, reproducible lockfiles. |
| Secrets | Pluggable `SecretStore` (env → Vault) | `data_sources.credentials_ref` points at the store, never raw credentials in DB (PCI/security). |

### Project Structure

```
ecommerce-analytics-platform/
├── pyproject.toml
├── uv.lock
├── Dockerfile
├── docker-compose.yml
├── .env.example
├── alembic.ini
├── README.md
├── openapi.json                      # generated, committed for partner SDKs
├── migrations/                       # Alembic
│   ├── env.py
│   └── versions/
├── dbt/                              # warehouse transformation marts
│   ├── dbt_project.yml
│   └── models/
│       ├── staging/
│       └── marts/{funnels,cohorts,attribution}.sql
├── src/eca/
│   ├── main.py                       # FastAPI app factory
│   ├── config.py                     # Pydantic Settings (env-driven)
│   ├── db/
│   │   ├── base.py                   # engine, session, Base
│   │   ├── models/                   # SQLAlchemy ORM (one file per domain)
│   │   └── partitions.py             # partition creation/maintenance helpers
│   ├── schemas/                      # Pydantic request/response + event specs
│   │   ├── events.py                 # Segment-spec event models
│   │   └── api/
│   ├── auth/                         # OIDC, JWT, RBAC dependencies
│   ├── tenancy/                      # tenant context middleware + RLS guard
│   ├── ingestion/
│   │   ├── pixel.py                  # first-party JS pixel collector endpoint
│   │   ├── http_events.py            # server-side Segment-compatible API
│   │   ├── ga4.py                    # GA4 Measurement Protocol normaliser
│   │   └── normaliser.py             # → canonical behavioural_events
│   ├── connectors/
│   │   ├── base.py                   # BaseConnector, SyncResult, cursors
│   │   ├── shopify.py  woocommerce.py  bigcommerce.py
│   │   ├── ads/{meta.py,google.py,tiktok.py}
│   │   └── registry.py
│   ├── analytics/
│   │   ├── metrics.py                # GMV, AOV, conv rate, CAC
│   │   ├── funnels.py                # funnel drop-off (SQL builders)
│   │   ├── cohorts.py                # retention + LTV
│   │   ├── attribution/             # last/first/linear/time-decay/position/prob
│   │   ├── inventory.py              # sell-through, days-of-stock, stockout
│   │   ├── creative.py               # creative-level ROAS
│   │   └── segments.py               # high-LTV/at-risk/lapsed segmentation
│   ├── identity/                     # session stitching + cross-device graph
│   ├── ml/
│   │   ├── forecasting.py            # Prophet/sklearn SKU demand
│   │   └── attribution_mmm.py        # Meridian wrapper
│   ├── ai/
│   │   ├── nl_query.py               # NL → SQL/metric plan → answer
│   │   ├── suggestions.py            # anomaly/stockout/creative suggestions
│   │   └── llm.py                    # provider-abstracted client
│   ├── mcp/server.py                 # MCP tool server
│   ├── alerts/                       # rules, evaluation, Slack/email delivery
│   ├── streaming/                    # StreamProducer (Redis Streams → Kafka)
│   ├── export/                       # Snowflake/BigQuery writers, dbt runner
│   ├── tasks/                        # Celery app, beat schedule, task defs
│   └── api/routes/                   # FastAPI routers per domain
├── frontend/                         # Next.js 16 app
│   ├── package.json
│   └── app/
└── tests/
    ├── unit/  integration/  e2e/  fixtures/
```

---

## Phase 1: Foundation — Project Scaffolding, Config, Multi-Tenancy

### Purpose
Establish the runnable skeleton: FastAPI app, settings, database connectivity, Docker Compose stack, CI quality gates, and the multi-tenant + RBAC primitives every later phase depends on. After this phase the app boots, serves a health check, runs migrations, and enforces tenant isolation on a trivial protected route.

### Tasks

#### 1.1 — Project scaffolding & tooling
**What**: Initialise the repo with uv, ruff, mypy, pytest, pre-commit, Dockerfile, and docker-compose for api/worker/beat/postgres/redis.

**Design**:
- `pyproject.toml` declares deps (fastapi, uvicorn, sqlalchemy>=2, alembic, pydantic>=2, pydantic-settings, celery[redis], redis, authlib, python-jose, httpx, structlog) and dev deps (pytest, pytest-asyncio, testcontainers, ruff, mypy).
- `config.py` — `Settings(BaseSettings)` with `database_url`, `redis_url`, `jwt_secret`, `oidc_issuer`, `oidc_client_id/secret`, `llm_provider`, `llm_api_key`, `secret_store` (`env|vault`), `environment` (`dev|prod`). All env-driven; `.env.example` documents each.
- `main.py` — `create_app()` factory mounts routers, exception handlers, structured-logging middleware, and `/healthz` returning `{"status":"ok","db":bool,"redis":bool}`.
- `docker-compose.yml` — services: `api`, `worker` (celery), `beat`, `postgres:16`, `redis:7`, `frontend`. Healthchecks gate `depends_on`.

**Testing**:
- Unit: `Settings` loads from env with defaults; missing `database_url` in prod → `ValidationError` naming the field.
- Integration: `GET /healthz` with PG+Redis up (testcontainers) → 200, both flags true; with Redis down → 200, `redis:false`.
- E2E: `docker compose up` then `curl /healthz` → 200.

#### 1.2 — Database base, migrations, partition helpers
**What**: SQLAlchemy `Base`/engine/session, Alembic configured, and helpers to create/maintain range partitions.

**Design**:
- `db/base.py` — async engine + `async_sessionmaker`; `get_session()` FastAPI dependency.
- `db/partitions.py` — `ensure_monthly_partition(table, month)` issues `CREATE TABLE IF NOT EXISTS <table>_yYYYYmMM PARTITION OF <table> FOR VALUES FROM (...) TO (...)`; `ensure_partitions_ahead(table, months=3)` for beat to call.
- Alembic `env.py` wired to `Settings.database_url` and `Base.metadata`.

**Testing**:
- Integration (real PG): create a partitioned parent, call `ensure_monthly_partition` → child exists; insert a row → lands in the correct child (verified via `tableoid`).
- Integration: calling the helper twice is idempotent (no error).

#### 1.3 — Tenancy & RBAC primitives
**What**: `tenants`/`users` tables, tenant-context middleware, and an RBAC dependency enforcing roles.

**Design** (DDL from Suggestion 1 Entity Management):
```sql
CREATE TABLE tenants ( id UUID PK DEFAULT gen_random_uuid(), name TEXT, slug TEXT UNIQUE,
  base_currency CHAR(3) DEFAULT 'USD', plan TEXT CHECK (plan IN ('free','starter','growth','enterprise')),
  timezone TEXT DEFAULT 'UTC', created_at TIMESTAMPTZ, updated_at TIMESTAMPTZ );
CREATE TABLE users ( id UUID PK, tenant_id UUID REFERENCES tenants(id), email TEXT, name TEXT,
  role TEXT CHECK (role IN ('owner','admin','analyst','viewer','api_service')),
  is_active BOOLEAN DEFAULT true, UNIQUE (tenant_id, email) );
```
- `tenancy/context.py` — `TenantContext(tenant_id, user_id, role)` resolved from JWT; stored in a `ContextVar`. A `tenant_scoped(query)` helper appends `WHERE tenant_id = :ctx` to every query; an `assert_tenant_match` guard rejects cross-tenant row access (defence-in-depth alongside optional PG RLS).
- `require_role(*roles)` FastAPI dependency → 403 if `ctx.role` not in set.

**Testing**:
- Unit: `require_role('admin')` with `viewer` ctx → `HTTPException(403)`.
- Integration: user in tenant A requests a tenant-B resource id → 404 (not 403, to avoid existence leak).
- Unit: `tenant_scoped` injects the predicate into a built statement.

---

## Phase 2: Canonical Data Model & Event Schemas

### Purpose
Materialise the full Suggestion-1 schema as ORM models + migrations and define the Pydantic event contracts (Segment E-Commerce Spec v2). This is the shared vocabulary all ingestion, analytics, and AI phases build on. After this phase the database holds every table and the canonical event types validate inbound payloads.

### Tasks

#### 2.1 — Commerce, product, inventory, customer, session, order, touchpoint models
**What**: ORM models + migration for `data_sources`, `channels`, `products`, `inventory_snapshots`, `customers`, `sessions`, `orders`, `order_items`, `touchpoints`, `attribution_results`, `behavioural_events`, `ai_suggestions`, `audit_log` exactly per Suggestion 1.

**Design**: Translate each `CREATE TABLE` from `data-model-suggestion-1.md` verbatim into SQLAlchemy models with the documented CHECK constraints, indexes (incl. partial + GIN), `UNIQUE` keys, and `PARTITION BY RANGE` for `sessions`(session_start), `orders`(order_date), `touchpoints`(occurred_at), `behavioural_events`(occurred_at), `audit_log`(created_at). Money stored as `*_cents BIGINT`; currency `CHAR(3)` (ISO 4217); country `CHAR(2)` (ISO 3166-1). Multi-currency orders keep `currency`, `total_cents`, plus `total_base_cents` + `fx_rate`.

**Testing**:
- Integration (real PG): run all migrations clean → all 15 tables + indexes exist (`information_schema`).
- Integration: inserting an `orders` row with `status='bogus'` → CHECK violation.
- Integration: duplicate `(tenant_id, data_source_id, external_id)` on products → unique violation.
- Migration round-trip: `alembic upgrade head` then `downgrade base` succeeds.

#### 2.2 — Canonical event schemas (Segment E-Commerce Spec v2)
**What**: Pydantic models for the canonical behavioural event taxonomy and the public event-ingest envelope.

**Design**:
```python
class EventContext(BaseModel):
    page_url: str | None; referrer: str | None; user_agent: str | None
    ip: str | None; device_type: Literal['desktop','mobile','tablet'] | None
    utm: dict[str,str] = {}; locale: str | None

class TrackEvent(BaseModel):
    anonymous_id: str | None; user_id: str | None  # external customer id
    event: Literal['page_viewed','product_viewed','product_list_viewed','product_added',
        'product_removed','cart_viewed','checkout_started','checkout_step_viewed',
        'checkout_step_completed','payment_info_entered','order_completed','order_refunded',
        'coupon_entered','coupon_applied','coupon_denied','promotion_viewed','promotion_clicked',
        'product_searched','product_shared','product_reviewed']
    timestamp: datetime; properties: dict[str,Any] = {}
    context: EventContext = EventContext()
    @model_validator: order_completed requires properties.order_id, total, currency, products[]
```
- `events.py` maps GA4 names (`view_item`→`product_viewed`, `add_to_cart`→`product_added`, `purchase`→`order_completed`) and Segment names → the canonical `event_name` CHECK set in `behavioural_events`.

**Testing**:
- Unit: `order_completed` missing `total` → ValidationError naming `total`.
- Unit: GA4 `purchase` → normalises to `order_completed`; `view_item`→`product_viewed`.
- Unit: unknown `event` value → ValidationError listing allowed names.

---

## Phase 3: Event Ingestion (Cookieless First-Party Capture)

### Purpose
Deliver the cookieless data foundation: a first-party JS pixel, a server-side HTTP event API (Segment-compatible), and a GA4 Measurement Protocol ingest, all normalising into `behavioural_events` + `sessions` with consent enforcement. This is the table-stakes capability without which no downstream analytics exist.

### Tasks

#### 3.1 — Server-side HTTP event API + normaliser
**What**: `POST /v1/events/track` and `POST /v1/events/batch` accepting Segment-spec payloads, authenticated by per-source write key, writing canonical events.

**Design**:
- Auth: `Authorization: Bearer <write_key>` resolves to a `data_sources` row (`source_type in ('segment','rudderstack','custom_api')`); rejects unknown keys (401).
- Batch limits mirror Segment: ≤500 KB/request, ≤32 KB/event, ≤100 events/batch → 413 on exceed.
- `normaliser.normalise(track, source)` → upserts `customers` (by `user_id`), resolves/creates `sessions` (by `anonymous_id` within session window = 30 min idle), writes `behavioural_events` (`source` column set). For `order_completed`, upserts `orders` + `order_items` and flips `sessions.converted=true`.
- Consent: events with `context.consent != 'opted_in'` for known EU IPs are stored only if a tenant config allows; otherwise dropped with a counter (GDPR — consent before analytics).

**Testing**:
- Integration (mocked auth, real PG): valid `product_viewed` → one `behavioural_events` row + a `sessions` row.
- Integration: `order_completed` → order + items rows, session `converted=true`.
- Integration: batch with a 40 KB event → 413, nothing written.
- Integration: unknown write key → 401.
- Unit: idle >30 min between two events from same `anonymous_id` → two sessions.

#### 3.2 — First-party JS pixel collector
**What**: A `/px` collector endpoint + a small served `pixel.js` that posts events same-origin (first-party, cookieless-capable).

**Design**: `pixel.js` exposes `eca.track(event, properties)` and auto-captures `page_viewed`; uses `localStorage` anonymous id (no third-party cookie). `GET /px.js` serves the script with the tenant write key injected; `POST /px` reuses the 3.1 normaliser. Sets `Cache-Control` and CORS to the tenant's allowed origins only.

**Testing**:
- Integration: `POST /px` with a valid origin → 204 + event stored; disallowed origin → 403.
- E2E (Playwright): load a test HTML page embedding `px.js`, trigger `eca.track('product_added')` → event appears in DB.

#### 3.3 — GA4 Measurement Protocol ingest
**What**: `POST /v1/events/ga4` accepting GA4 MP payloads, normalising the `items` array and event names.

**Design**: Validates `measurement_id` + `api_secret` against a `data_sources` row (`source_type='ga4'`); ≤25 events/request (GA4 limit); maps each `events[].name` via the Phase 2 GA4 map; expands `items[]` into product references. Rejects events older than 48h (GA4 joining window).

**Testing**:
- Unit: GA4 `purchase` with `items` → canonical `order_completed` + order_items.
- Integration: 26 events → 400 (limit). Event timestamped 49h ago → dropped with reason.

---

## Phase 4: Commerce Connectors (Shopify, WooCommerce, BigCommerce)

### Purpose
Backfill and keep current the orders, products, customers, and inventory that anchor every metric. Native connectors (vs. Airbyte) give control over webhook latency needed for real-time inventory alerts later. After this phase a merchant connects a store via OAuth and sees their historical orders and live products in the platform.

### Tasks

#### 4.1 — Connector framework
**What**: `BaseConnector` abstraction with OAuth, cursored incremental sync, and idempotent upserts.

**Design**:
```python
class SyncResult(BaseModel):
    records: int; cursor: str | None; errors: list[str]
class BaseConnector(ABC):
    source_type: str
    def authorize_url(self, redirect_uri) -> str: ...
    async def exchange_code(self, code) -> Credentials: ...      # → SecretStore, ref in data_sources
    async def sync_products(self, since_cursor) -> SyncResult: ...
    async def sync_orders(self, since_cursor) -> SyncResult: ...
    async def sync_customers(self, since_cursor) -> SyncResult: ...
    async def sync_inventory(self) -> SyncResult: ...
```
- OAuth 2.0 authorization-code flow per standards.md; credentials stored via `SecretStore`, only `credentials_ref` in `data_sources`.
- Upserts key on `(tenant_id, data_source_id, external_id)`; sync cursor persisted to `data_sources.sync_cursor`; status transitions `pending→connected→syncing→active|error`.
- Celery task `run_sync(data_source_id, entity)`; beat schedules incremental syncs every 15 min.

**Testing**:
- Unit: status state machine rejects illegal transitions (e.g. `disconnected→syncing`).
- Integration (mocked HTTP): re-running a sync with the same records → upsert, no duplicates.
- Unit: failed exchange → status `error`, message recorded.

#### 4.2 — Shopify connector (GraphQL Admin + webhooks)
**What**: Backfill via GraphQL Admin API + register webhooks for live order/product/inventory changes.

**Design**: GraphQL bulk operations for historical backfill (orders, products, customers, inventory levels); maps Shopify order → `orders`+`order_items` (cents, currency, fx, COGS from `unitCost`); registers webhooks (`orders/create`, `orders/updated`, `products/update`, `inventory_levels/update`) verified by HMAC-SHA256 against the app secret. Inventory webhook writes an `inventory_snapshots` upsert for the day and emits a `StreamProducer` inventory-change event (used in Phase 9).

**Testing**:
- Integration (mocked Shopify): backfill fixture of 50 orders → 50 orders + correct line items, COGS/margin populated.
- Integration: webhook with valid HMAC → processed; tampered body → 401, no write.
- Unit: Shopify multi-currency order → `total_base_cents` computed from `fx_rate`.

#### 4.3 — WooCommerce & BigCommerce connectors
**What**: REST-based connectors implementing the same `BaseConnector` interface.

**Design**: WooCommerce REST (`/wp-json/wc/v3`, basic/consumer-key auth) and BigCommerce (`X-Auth-Token`); paginate via cursor/page; map each platform's order/product/customer schema into the canonical tables. Webhook endpoints where available; otherwise polling sync.

**Testing**:
- Integration (mocked): Woo + BigCommerce order fixtures → canonical orders identical in shape to Shopify's, proving schema reconciliation.
- Fixture: catalogue with categories → `products.category/subcategory` populated for funnel segmentation.

---

## Phase 5: Core Analytics Engine (Metrics, Funnels, Cohorts, Attribution)

### Purpose
Ship the heart of the product: the MVP analytical computations exposed as APIs. After this phase the platform answers the MVP questions — GMV/AOV/conversion/CAC by channel, funnel drop-off by source/device/category, cohort retention + LTV, and multi-touch attribution (last/first/linear).

### Tasks

#### 5.1 — Core commerce metrics
**What**: `GET /v1/metrics` returning GMV, AOV, conversion rate, CAC, and contribution margin for a date range with grouping.

**Design**:
```
GET /v1/metrics?from=&to=&group_by=channel|device|category|day&metrics=gmv,aov,conv_rate,cac,margin
```
- GMV = Σ `orders.total_base_cents` (status not in cancelled/refunded); AOV = GMV/order_count; conversion rate = converted_sessions/sessions; CAC = channel spend (Σ `touchpoints.cost_cents`) / new customers acquired on that channel; margin = Σ `order_items.margin_cents`.
- Implemented as parameterised SQL builders (SQLAlchemy Core) with partition-pruned date predicates; results cached in Redis keyed by `(tenant, query_hash)` TTL 5 min.
- Response is an OpenAPI-typed `MetricSeries` (rows of `{group, metric, value, currency}`).

**Testing**:
- Integration (seeded PG): known order set → exact GMV/AOV asserted.
- Integration: `group_by=channel` → CAC per channel matches hand-computed values.
- Unit: refunded orders excluded from GMV.
- Integration: cache hit returns identical payload without a second DB hit (spy).

#### 5.2 — Conversion funnel analysis
**What**: `GET /v1/funnels` computing step-to-step drop-off, segmentable by source/device/category.

**Design**: Caller supplies an ordered step list of canonical event names (default: `product_viewed → product_added → checkout_started → order_completed`). For each step, count distinct sessions reaching it within the window; compute conversion and drop-off rates; segment by `sessions.channel_id`/`device_type` or product `category`. SQL uses `FILTER (WHERE event_name=...)` over `behavioural_events` joined to `sessions`.

**Testing**:
- Integration: seeded journeys where 100 view, 40 add, 25 checkout, 18 buy → exact step counts and drop-off %.
- Integration: `segment_by=device_type` splits counts correctly.
- Edge: a step no session reaches → 0 with no division error.

#### 5.3 — Cohort retention & LTV
**What**: `GET /v1/cohorts` returning a retention matrix and LTV by acquisition month.

**Design**: Cohort = `customers.cohort_month`. For month offsets 0..N, fraction of the cohort placing a repeat order in each subsequent month; LTV = cumulative Σ `orders.total_base_cents` per cohort / cohort size. Returns a matrix `{cohort_month, offsets:[{n, retained_pct, cumulative_ltv_cents}]}` for a heatmap.

**Testing**:
- Integration: 3-month synthetic cohort with known repeat behaviour → retention matrix matches.
- Unit: cohort with zero repeat purchases → all offsets 0% except month 0.

#### 5.4 — Multi-touch attribution (last/first/linear)
**What**: Attribution engine computing per-model credit and persisting `attribution_results`.

**Design**:
```python
class AttributionModel(Protocol):
    name: str
    def assign(self, touchpoints: list[Touchpoint], order: Order) -> list[CreditAssignment]
# last_click: 100% to final pre-conversion touchpoint
# first_click: 100% to first
# linear: equal split across all touchpoints in the lookback window (default 30d)
```
- For each converting order, gather the customer's touchpoints inside the lookback; each model emits `credit_weight` rows (Σ=1.0) into `attribution_results` with `attributed_revenue_cents = weight * order.total_base_cents` and `days_to_conversion`.
- Recompute is a Celery task; `GET /v1/attribution?model=&from=&to=` aggregates `attributed_revenue_cents` and ROAS (revenue/spend) by channel.

**Testing**:
- Unit: order with touchpoints [Meta, Google, Email] → last_click gives Email 100%; first_click Meta 100%; linear 33.3% each (Σ=1.0).
- Integration: side-by-side model comparison returns differing channel revenue from identical touchpoints.
- Edge: order with no touchpoints in lookback → attributed to `direct` channel.

---

## Phase 6: Auth, API Surface Hardening & OpenAPI Publication

### Purpose
Make the platform safe to expose: OIDC login + JWT API auth, per-tenant API keys, rate limiting, OWASP hardening, and a published OpenAPI 3.1 document with the audit log populated. After this phase external partners can authenticate and generate SDKs from the spec.

### Tasks

#### 6.1 — OIDC user auth + API keys + RBAC enforcement
**What**: OIDC login flow, JWT issuance, per-tenant API keys for the `api_service` role, RBAC across all routes.

**Design**: Authlib OIDC against `Settings.oidc_issuer` (authorization-code + PKCE); on callback, upsert `users`, issue a short-lived JWT (tenant_id, user_id, role) + refresh token. API keys are hashed (argon2) and resolve to an `api_service` principal scoped to one tenant. Every route declares `Depends(require_role(...))`.

**Testing**:
- Integration (mocked IdP): valid OIDC callback → user upserted, JWT returned.
- Unit: expired JWT → 401; tampered signature → 401.
- Integration: viewer hitting a write route → 403.

#### 6.2 — Rate limiting, input hardening, audit log
**What**: Redis token-bucket rate limiting, request-size/validation guards, and `audit_log` writes on mutations.

**Design**: Per-key/per-IP token buckets (default 600 req/min, ingest endpoints higher); 429 with `Retry-After`. Pydantic strict mode + max body sizes guard injection/oversized payloads (OWASP A03/A04). A mutation dependency writes `audit_log` (actor_type/id, action, resource_type/id, changes, ip, user_agent). PCI guard: middleware rejects any payload containing a PAN-shaped value (Luhn-valid 13–19 digit) on ingest endpoints.

**Testing**:
- Integration: 601 requests in a minute → 429 with `Retry-After`.
- Integration: a connector disconnect → `audit_log` row with correct actor/action.
- Unit: payload containing a Luhn-valid card number → 422, nothing stored (PCI).

#### 6.3 — OpenAPI publication
**What**: Generate, validate, and commit `openapi.json` (OAS 3.1) covering all routes.

**Design**: FastAPI auto-generates the schema; a build step writes `openapi.json` and validates it against the OAS 3.1 meta-schema in CI; webhook endpoints described via OAS webhooks. Drift between code and committed spec fails CI.

**Testing**:
- Unit: generated schema validates against OAS 3.1 meta-schema.
- CI: stale committed `openapi.json` vs. live app → build fails.

---

## Phase 7: Merchant Dashboard (Frontend)

### Purpose
Give merchants the UX patterns the incumbents win on: a daily metric-tile summary, attribution waterfall with period comparison, funnel view, and cohort heatmap. After this phase a non-SQL user navigates their analytics in the browser.

### Tasks

#### 7.1 — App shell, auth, tenant switcher
**What**: Next.js 16 App Router shell with OIDC login, JWT handling, and tenant/role-aware navigation.

**Design**: Server Components fetch via the typed API client generated from `openapi.json`; auth via the Phase 6 OIDC flow; role gates hide write actions from viewers; shadcn/ui + Tailwind layout with date-range picker driving all dashboards.

**Testing**:
- E2E (Playwright): unauthenticated user → redirected to login; after login → dashboard renders.
- E2E: viewer role sees no "Connect source" button.

#### 7.2 — Metrics, funnel, cohort, attribution views
**What**: Four dashboards bound to the Phase 5 APIs.

**Design**: Summary tiles (GMV, AOV, conv rate, CAC, margin) with period-over-period deltas; attribution waterfall (Recharts) with model selector and PoP comparison; funnel bar/step chart with source/device segment toggles; cohort retention heatmap. All respect the global date range and tenant context.

**Testing**:
- E2E: pick a date range → tiles update; switch attribution model → waterfall re-renders with different channel split.
- Component: funnel renders correct bar heights from a fixture payload.

---

## Phase 8: Inventory Performance, Creative Analytics, Segmentation, Warehouse Export, Probabilistic Attribution (v1.1)

### Purpose
Deliver the headline differentiators that incumbents lack or silo: supply-side inventory analytics joined to marketing, creative-level ROAS, exportable customer segments, warehouse export for SQL teams, and probabilistic attribution. These extend the engine additively.

### Tasks

#### 8.1 — Inventory performance & stockout detection
**What**: Sell-through rate, days-of-stock, and stockout flags from `inventory_snapshots`, exposed via `GET /v1/inventory`.

**Design**: Daily Celery job computes per product: `sell_through_rate = units_sold_period / (units_sold_period + quantity_on_hand)`; `days_of_stock = quantity_available / avg_daily_units_sold`; `is_stockout = quantity_available <= 0`. Stockout-vs-spend join surfaces SKUs that are out of stock yet have active `touchpoints.cost_cents` in the period (the core demand/supply differentiator). Window functions over the daily snapshot series.

**Testing**:
- Integration: product selling 5/day with 10 on hand → days_of_stock=2.0.
- Integration: out-of-stock SKU with ad spend in window → appears in the stockout-vs-spend report.

#### 8.2 — Creative-level paid-media analytics
**What**: `GET /v1/creative` ranking ad creatives by ROAS, conversions, and spend.

**Design**: Group `attribution_results` joined to `touchpoints` by `creative_id`/`creative_name`; ROAS = attributed_revenue / creative spend; thumb-stop/engagement carried in `touchpoints` metrics where the ad connector provides them. Sortable, filterable by channel/date.

**Testing**:
- Integration: two creatives with known spend + attributed revenue → correct ROAS ranking.

#### 8.3 — Customer segmentation with exportable lists
**What**: Recompute `customers.segment` (new/active/high_ltv/at_risk/lapsed/churned) and expose `GET /v1/segments/{segment}/export` (CSV).

**Design**: Rule engine: high_ltv = LTV ≥ tenant P90; at_risk = days since last order > 2× median inter-order interval; lapsed/churned by configurable day thresholds; runs nightly via beat. Export streams a CSV of consenting customers only (GDPR).

**Testing**:
- Unit: customer above P90 LTV → high_ltv.
- Integration: export excludes `consent_status='opted_out'` customers.

#### 8.4 — Warehouse export (Snowflake/BigQuery) + dbt marts
**What**: Scheduled export of canonical tables + dbt-built marts to Snowflake/BigQuery.

**Design**: `export/` writers batch-load partitioned tables incrementally (watermark per table) into the destination; dbt-core models build `funnels`, `cohorts`, `attribution` marts in the warehouse for downstream BI (Looker/Metabase/Superset). Export config + credentials via `SecretStore`.

**Testing**:
- Integration (mocked warehouse client): incremental export sends only rows after the watermark; watermark advances.
- Unit: dbt model SQL compiles (dbt parse) against the mart schema.

#### 8.5 — Probabilistic attribution (Meridian MMM)
**What**: A `probabilistic` model correcting for platform under-reporting, persisted alongside deterministic models.

**Design**: `ml/attribution_mmm.py` wraps Google Meridian: fits a media-mix model on channel spend + conversions to estimate channel contribution; outputs blended channel weights written as `attribution_results` rows with `model='probabilistic'`, comparable side-by-side in the Phase 7 waterfall. Trained on a Celery schedule.

**Testing**:
- Integration (small synthetic spend/conversion series): model fits and emits probabilistic weights summing sensibly across channels.
- Unit: Meridian wrapper handles a single-channel input without error.

---

## Phase 9: AI-Native Layer (NL Analytics, Suggestions, MCP, Real-Time Inventory-to-Ad-Spend)

### Purpose
Deliver the AI-native advantage: natural-language analytics, proactive AI suggestions, an MCP server for external agents, and real-time pausing of campaigns for out-of-stock SKUs. These close the loop between measurement and action.

### Tasks

#### 9.1 — Natural-language analytics
**What**: `POST /v1/ask` answering questions like "why did ROAS drop on Meta last week?" without SQL.

**Design**: `nl_query.py` is a constrained planner — the LLM maps the question to one of the existing analytics functions (metrics/funnels/cohorts/attribution/inventory/creative) with parameters (a tool-calling schema), never raw SQL against the DB, to bound risk. Results are summarised by the LLM with the underlying figures cited. Tenant-scoped; respects RBAC.

**Testing**:
- Unit (mocked LLM): "GMV last 7 days" → plan calls `metrics(metric=gmv, from=−7d)`; answer embeds the returned number.
- Integration: a question outside scope → graceful "I can't answer that yet" rather than fabricated SQL.

#### 9.2 — AI suggestions engine
**What**: Generate `ai_suggestions` (roas_anomaly, creative_underperforming, stockout_risk, churn_risk, campaign_pause) with evidence and recommended actions.

**Design**: Scheduled detectors compute statistical anomalies (ROAS week-over-week z-score, creative below cohort median, stockout risk from days_of_stock < lead time); each writes an `ai_suggestions` row with `evidence` JSONB, `confidence`, `severity`, lifecycle `pending→accepted|dismissed|expired|auto_applied`. Surfaced in the dashboard for accept/dismiss.

**Testing**:
- Integration: a channel with a sharp ROAS drop → a `roas_anomaly` suggestion with evidence.
- Unit: dismissing a suggestion sets `status='dismissed'` + `resolved_at`.

#### 9.3 — Real-time inventory-to-ad-spend automation
**What**: When a SKU goes out of stock, emit a `campaign_pause` suggestion (and optionally auto-pause via ad connector).

**Design**: Shopify/Woo inventory webhooks (Phase 4) push to `StreamProducer` (Redis Streams); a consumer matches the SKU to active campaigns with spend and creates a `campaign_pause` suggestion; if the tenant enabled auto-apply, the ad connector pauses the campaign and the suggestion is marked `auto_applied` (audited). Stream abstraction allows Kafka swap at scale.

**Testing**:
- Integration: inventory webhook driving a SKU to 0 with an active campaign → `campaign_pause` suggestion created within the consumer loop.
- Integration (auto-apply on, mocked ad API): suggestion → pause call made, status `auto_applied`, audit row written.

#### 9.4 — MCP server
**What**: An MCP server exposing `query_sales_metrics`, `get_funnel_report`, `get_product_performance` to external AI clients.

**Design**: `mcp/server.py` implements the MCP tools per standards.md, each delegating to the Phase 5/8 analytics functions, authenticated by a tenant-scoped API key; read-only.

**Testing**:
- Integration: MCP `query_sales_metrics` for a range → same numbers as `GET /v1/metrics`.
- Unit: missing/invalid API key → tool returns an auth error, no data.

---

## Phase 10: SKU Demand Forecasting, Cross-Device Identity, Marketplace Attribution (backlog)

### Purpose
Complete the differentiating backlog: predictive reorder alerting, cookieless cross-device identity resolution, and unified marketplace attribution across owned store + Amazon + social.

### Tasks

#### 10.1 — SKU-level demand forecasting & reorder alerting
**What**: Forecast per-SKU demand and emit reorder suggestions when stock will breach the reorder point.

**Design**: `ml/forecasting.py` trains Prophet (seasonal) / sklearn (regression with external regressors) per high-velocity SKU on the `inventory_snapshots` + order history series; predicts days to stockout; when `predicted_days_to_stockout < lead_time`, writes a `demand_forecast`/`stockout_risk` suggestion with reorder quantity. Models retrained nightly; degrades gracefully for sparse/seasonal SKUs by falling back to moving-average.

**Testing**:
- Integration: SKU with a clear upward trend → forecast predicts earlier stockout than flat baseline.
- Unit: SKU with <14 days of history → moving-average fallback, no crash.

#### 10.2 — Cross-device identity resolution
**What**: Stitch sessions across devices using the first-party login graph (no third-party cookies).

**Design**: `identity/` builds an identity graph keyed on shared `user_id` (login) and deterministic signals (hashed email); merges `anonymous_id`s seen before/after identification into a single customer, rewriting `sessions.customer_id` and re-running attribution for affected orders. Probabilistic matching is opt-in and consent-gated.

**Testing**:
- Integration: anonymous mobile session then desktop login with the same `user_id` → both sessions linked to one customer; attribution re-computed.
- Unit: opted-out customer excluded from probabilistic matching.

#### 10.3 — Marketplace attribution (owned store + Amazon + social)
**What**: Reconcile Amazon Seller and social-commerce orders into the canonical model with consistent margin.

**Design**: Amazon Seller Central + social connectors (via Airbyte where native is heavy) map marketplace orders/fees into `orders` (`data_source.category='commerce'`) with marketplace fee handling so margin is comparable across channels; attribution treats marketplace as a `channel_type='marketplace'`. Unified GMV/margin views span all sources.

**Testing**:
- Integration: Amazon order fixture with fees → canonical order with margin net of marketplace fees.
- Integration: blended GMV across owned + marketplace sums correctly without double counting.

---

## Phase Summary & Dependencies

```
Phase 1: Foundation (tenancy, config, docker)         ─── required by everything
    │
Phase 2: Data Model & Event Schemas                   ─── requires P1
    │
Phase 3: Event Ingestion (cookieless)        ┐
Phase 4: Commerce Connectors                 ┘ ─── both require P2; can parallel
    │
Phase 5: Core Analytics Engine (MVP heart)            ─── requires P3 + P4
    │
Phase 6: Auth / Hardening / OpenAPI          ┐ ─── requires P5 (depends on P1 auth stubs)
Phase 7: Merchant Dashboard                  ┘ ─── requires P5 + P6; FE can parallel P8 backend
    │
Phase 8: Inventory / Creative / Segments /            ─── requires P5 (+P4 for inventory, +P6 for export auth)
         Warehouse Export / Probabilistic attr
    │
Phase 9: AI Layer (NL, suggestions, MCP,              ─── requires P5 + P8 (+P4 webhooks for 9.3)
         real-time inventory-to-ad-spend)
    │
Phase 10: Forecasting / Cross-device / Marketplace    ─── requires P8 (+P9 for suggestion surfacing)
```

**Parallelism opportunities**
- Phases 3 and 4 can be developed concurrently once Phase 2 lands (shared canonical tables, independent ingestion paths).
- Phase 7 (frontend) can be built against Phase 5/6 APIs in parallel with Phase 8 backend work.
- Within Phase 8, tasks 8.1–8.5 are mutually independent after Phase 5 and can be split across developers.
- Within Phase 10, 10.1/10.2/10.3 are independent.

---

## Definition of Done (per phase)

1. All tasks in the phase implemented and wired into the app/worker.
2. All unit and integration tests pass; integration tests run against real PostgreSQL + Redis via testcontainers.
3. `ruff check` + `ruff format --check` pass; `mypy --strict` passes (frontend: ESLint + Prettier).
4. `docker compose build` succeeds and the affected services start healthy.
5. The phase's feature works end-to-end (E2E/Playwright test green where user-facing).
6. New configuration options added to `config.py` and documented in `.env.example`.
7. New API endpoints appear in the regenerated, committed `openapi.json` (CI fails on drift).
8. New tables/columns have an Alembic migration that upgrades and downgrades cleanly; new partitioned tables register with the partition-maintenance beat job.
9. Mutations write `audit_log` rows; data handling respects consent (`consent_status`) and never stores payment-card data (PCI).
10. Tenant isolation verified: no endpoint or query can read another tenant's rows.
```
