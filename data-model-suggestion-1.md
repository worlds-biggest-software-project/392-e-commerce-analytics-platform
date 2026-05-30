# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: E-Commerce Analytics Platform · Created: 2026-05-29

## Philosophy

This model gives every e-commerce analytics concept its own table with explicit foreign keys. The data flows through a clear pipeline: commerce platform connectors import orders, products, and customers; first-party event capture writes behavioural events; attribution logic links sessions to purchases through touchpoints; and inventory snapshots track stock levels alongside marketing spend. Every relationship — from ad click to session to cart to order to product — is an explicit join, making the full shopper journey queryable in standard SQL.

The schema mirrors the Segment E-Commerce Spec v2 event taxonomy (Product Viewed, Cart Viewed, Checkout Step Viewed, Order Completed) while storing each concept relationally rather than as raw events. Multi-touch attribution models (last-click, first-click, linear, probabilistic) are computed across normalised touchpoint and session tables, and inventory performance metrics (sell-through rate, days-of-stock) are derived from dedicated product and inventory tables.

**Best for:** Data teams that want full SQL access to every dimension of e-commerce analytics, need to join behavioural events with inventory and financial data in a single query, and require referential integrity for regulatory compliance and financial reporting.

**Trade-offs:**
- (+) Full referential integrity from ad impression → touchpoint → session → cart → order → product
- (+) Standard SQL for all analytics — no JSONB operators or event replay required
- (+) Clear audit trail for attribution: every touchpoint and its weight are stored explicitly
- (+) Inventory and marketing data joined naturally via product foreign keys
- (-) Higher table count increases migration complexity as the platform evolves
- (-) High-volume behavioural events (page views, product views) require partitioned tables
- (-) Schema changes needed when adding new commerce platforms or ad channels

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| Segment E-Commerce Spec v2 | Event categories and property names align with Segment's Product Viewed, Cart Viewed, Order Completed taxonomy |
| GA4 Measurement Protocol | Event ingestion normalises GA4 event names (view_item, add_to_cart, purchase) into canonical categories |
| W3C Attribution Reporting API | Privacy-preserving attribution signals integrated alongside first-party touchpoint data |
| ISO 4217 | Currency codes for multi-currency order and revenue reporting |
| ISO 3166-1 | Country codes for geographic segmentation and shipping analytics |
| OpenAPI 3.2 | REST API surface documented per OAS 3.2 with JSON Schema 2020-12 |
| OAuth 2.0 | Platform connector authorisation for Shopify, Meta Ads, Google Ads |
| PCI DSS v4.0 | No raw card data stored; only tokenised payment references from processors |
| GDPR / CCPA | Consent-aware event capture; customer deletion cascades with aggregate preservation |

---

## Entity Management

```sql
CREATE TABLE tenants (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL UNIQUE,
    base_currency   CHAR(3) NOT NULL DEFAULT 'USD',
    plan            TEXT NOT NULL CHECK (plan IN ('free', 'starter', 'growth', 'enterprise')),
    timezone        TEXT NOT NULL DEFAULT 'UTC',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    email           TEXT NOT NULL,
    name            TEXT NOT NULL,
    role            TEXT NOT NULL CHECK (role IN ('owner', 'admin', 'analyst', 'viewer', 'api_service')),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, email)
);

CREATE INDEX idx_users_tenant ON users(tenant_id);
```

## Commerce Connectors & Channels

```sql
CREATE TABLE data_sources (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    name            TEXT NOT NULL,
    source_type     TEXT NOT NULL CHECK (source_type IN (
        'shopify', 'woocommerce', 'bigcommerce', 'amazon_seller',
        'meta_ads', 'google_ads', 'tiktok_ads', 'pinterest_ads',
        'klaviyo', 'segment', 'rudderstack', 'ga4',
        'custom_api', 'csv_import'
    )),
    category        TEXT NOT NULL CHECK (category IN ('commerce', 'advertising', 'analytics', 'marketing', 'custom')),
    status          TEXT NOT NULL DEFAULT 'pending' CHECK (status IN (
        'pending', 'connected', 'syncing', 'active', 'error', 'disconnected'
    )),
    credentials_ref TEXT,
    last_sync_at    TIMESTAMPTZ,
    sync_cursor     TEXT,
    config          JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_data_sources_tenant ON data_sources(tenant_id);

CREATE TABLE channels (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    name            TEXT NOT NULL,
    channel_type    TEXT NOT NULL CHECK (channel_type IN (
        'paid_search', 'paid_social', 'organic_search', 'organic_social',
        'email', 'sms', 'direct', 'referral', 'affiliate',
        'marketplace', 'display', 'video', 'other'
    )),
    data_source_id  UUID REFERENCES data_sources(id),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, name)
);

CREATE INDEX idx_channels_tenant ON channels(tenant_id);
```

## Products & Inventory

```sql
CREATE TABLE products (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    data_source_id  UUID NOT NULL REFERENCES data_sources(id),
    external_id     TEXT NOT NULL,
    sku             TEXT,
    name            TEXT NOT NULL,
    category        TEXT,
    subcategory     TEXT,
    brand           TEXT,
    price_cents     BIGINT NOT NULL,
    cost_cents      BIGINT,
    currency        CHAR(3) NOT NULL,
    image_url       TEXT,
    product_url     TEXT,
    status          TEXT NOT NULL DEFAULT 'active' CHECK (status IN ('active', 'archived', 'out_of_stock')),
    tags            TEXT[] NOT NULL DEFAULT '{}',
    attributes      JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, data_source_id, external_id)
);

CREATE INDEX idx_products_tenant ON products(tenant_id);
CREATE INDEX idx_products_sku ON products(tenant_id, sku);
CREATE INDEX idx_products_category ON products(tenant_id, category);
CREATE INDEX idx_products_tags ON products USING GIN (tags);

CREATE TABLE inventory_snapshots (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    product_id      UUID NOT NULL REFERENCES products(id),
    snapshot_date   DATE NOT NULL,
    quantity_on_hand INT NOT NULL,
    quantity_reserved INT NOT NULL DEFAULT 0,
    quantity_available INT NOT NULL,
    reorder_point   INT,
    days_of_stock   NUMERIC(8,2),
    sell_through_rate NUMERIC(8,4),
    units_sold_period INT NOT NULL DEFAULT 0,
    is_stockout     BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, product_id, snapshot_date)
);

CREATE INDEX idx_inventory_tenant_date ON inventory_snapshots(tenant_id, snapshot_date);
CREATE INDEX idx_inventory_product ON inventory_snapshots(product_id);
CREATE INDEX idx_inventory_stockout ON inventory_snapshots(tenant_id, snapshot_date) WHERE is_stockout;
```

## Customers & Sessions

```sql
CREATE TABLE customers (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    data_source_id  UUID NOT NULL REFERENCES data_sources(id),
    external_id     TEXT NOT NULL,
    email           TEXT,
    name            TEXT,
    country         CHAR(2),
    city            TEXT,
    first_seen_at   TIMESTAMPTZ NOT NULL,
    first_order_at  TIMESTAMPTZ,
    last_order_at   TIMESTAMPTZ,
    total_orders    INT NOT NULL DEFAULT 0,
    total_revenue_cents BIGINT NOT NULL DEFAULT 0,
    ltv_cents       BIGINT NOT NULL DEFAULT 0,
    aov_cents       BIGINT,
    acquisition_channel TEXT,
    cohort_month    DATE,
    segment         TEXT CHECK (segment IN ('new', 'active', 'high_ltv', 'at_risk', 'lapsed', 'churned')),
    custom_attributes JSONB NOT NULL DEFAULT '{}',
    tags            TEXT[] NOT NULL DEFAULT '{}',
    consent_status  TEXT NOT NULL DEFAULT 'unknown' CHECK (consent_status IN (
        'opted_in', 'opted_out', 'unknown'
    )),
    is_deleted      BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, data_source_id, external_id)
);

CREATE INDEX idx_customers_tenant ON customers(tenant_id);
CREATE INDEX idx_customers_segment ON customers(tenant_id, segment);
CREATE INDEX idx_customers_cohort ON customers(tenant_id, cohort_month);
CREATE INDEX idx_customers_tags ON customers USING GIN (tags);

CREATE TABLE sessions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    customer_id     UUID REFERENCES customers(id),
    anonymous_id    TEXT,
    session_start   TIMESTAMPTZ NOT NULL,
    session_end     TIMESTAMPTZ,
    duration_seconds INT,
    device_type     TEXT CHECK (device_type IN ('desktop', 'mobile', 'tablet')),
    browser         TEXT,
    os              TEXT,
    country         CHAR(2),
    city            TEXT,
    landing_page    TEXT,
    exit_page       TEXT,
    page_views      INT NOT NULL DEFAULT 0,
    utm_source      TEXT,
    utm_medium      TEXT,
    utm_campaign    TEXT,
    utm_content     TEXT,
    utm_term        TEXT,
    referrer        TEXT,
    channel_id      UUID REFERENCES channels(id),
    converted       BOOLEAN NOT NULL DEFAULT false,
    order_id        UUID,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (session_start);

CREATE INDEX idx_sessions_tenant ON sessions(tenant_id, session_start);
CREATE INDEX idx_sessions_customer ON sessions(customer_id);
CREATE INDEX idx_sessions_channel ON sessions(channel_id);
CREATE INDEX idx_sessions_converted ON sessions(tenant_id) WHERE converted;
```

## Orders & Line Items

```sql
CREATE TABLE orders (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    data_source_id  UUID NOT NULL REFERENCES data_sources(id),
    customer_id     UUID NOT NULL REFERENCES customers(id),
    external_id     TEXT NOT NULL,
    order_number    TEXT,
    order_date      TIMESTAMPTZ NOT NULL,
    status          TEXT NOT NULL CHECK (status IN (
        'pending', 'confirmed', 'fulfilled', 'shipped', 'delivered',
        'cancelled', 'refunded', 'partially_refunded'
    )),
    subtotal_cents  BIGINT NOT NULL,
    discount_cents  BIGINT NOT NULL DEFAULT 0,
    shipping_cents  BIGINT NOT NULL DEFAULT 0,
    tax_cents       BIGINT NOT NULL DEFAULT 0,
    total_cents     BIGINT NOT NULL,
    currency        CHAR(3) NOT NULL,
    total_base_cents BIGINT NOT NULL,
    fx_rate         NUMERIC(18,8) NOT NULL DEFAULT 1.0,
    payment_method  TEXT,
    shipping_country CHAR(2),
    coupon_code     TEXT,
    is_first_order  BOOLEAN NOT NULL DEFAULT false,
    session_id      UUID REFERENCES sessions(id),
    attribution_channel_id UUID REFERENCES channels(id),
    tags            TEXT[] NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, data_source_id, external_id)
) PARTITION BY RANGE (order_date);

CREATE INDEX idx_orders_tenant_date ON orders(tenant_id, order_date);
CREATE INDEX idx_orders_customer ON orders(customer_id);
CREATE INDEX idx_orders_channel ON orders(attribution_channel_id);
CREATE INDEX idx_orders_status ON orders(tenant_id, status);

CREATE TABLE order_items (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_id        UUID NOT NULL REFERENCES orders(id),
    product_id      UUID NOT NULL REFERENCES products(id),
    quantity        INT NOT NULL,
    unit_price_cents BIGINT NOT NULL,
    discount_cents  BIGINT NOT NULL DEFAULT 0,
    total_cents     BIGINT NOT NULL,
    cost_cents      BIGINT,
    margin_cents    BIGINT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_order_items_order ON order_items(order_id);
CREATE INDEX idx_order_items_product ON order_items(product_id);
```

## Attribution & Touchpoints

```sql
CREATE TABLE touchpoints (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    customer_id     UUID REFERENCES customers(id),
    anonymous_id    TEXT,
    session_id      UUID REFERENCES sessions(id),
    channel_id      UUID REFERENCES channels(id),
    touchpoint_type TEXT NOT NULL CHECK (touchpoint_type IN (
        'ad_click', 'ad_view', 'organic_click', 'email_click',
        'sms_click', 'referral_click', 'direct_visit', 'social_click'
    )),
    occurred_at     TIMESTAMPTZ NOT NULL,
    campaign_id     TEXT,
    campaign_name   TEXT,
    ad_group_id     TEXT,
    ad_group_name   TEXT,
    creative_id     TEXT,
    creative_name   TEXT,
    keyword         TEXT,
    landing_page    TEXT,
    cost_cents      BIGINT,
    impressions     INT,
    clicks          INT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (occurred_at);

CREATE INDEX idx_touchpoints_tenant ON touchpoints(tenant_id, occurred_at);
CREATE INDEX idx_touchpoints_customer ON touchpoints(customer_id);
CREATE INDEX idx_touchpoints_channel ON touchpoints(channel_id);
CREATE INDEX idx_touchpoints_creative ON touchpoints(tenant_id, creative_id);

CREATE TABLE attribution_results (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    order_id        UUID NOT NULL REFERENCES orders(id),
    touchpoint_id   UUID NOT NULL REFERENCES touchpoints(id),
    channel_id      UUID NOT NULL REFERENCES channels(id),
    model           TEXT NOT NULL CHECK (model IN (
        'last_click', 'first_click', 'linear', 'time_decay',
        'position_based', 'probabilistic', 'data_driven'
    )),
    credit_weight   NUMERIC(8,6) NOT NULL,  -- 0.0 to 1.0
    attributed_revenue_cents BIGINT NOT NULL,
    attributed_margin_cents  BIGINT,
    days_to_conversion INT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_attribution_order ON attribution_results(order_id);
CREATE INDEX idx_attribution_channel ON attribution_results(tenant_id, channel_id, model);
CREATE INDEX idx_attribution_touchpoint ON attribution_results(touchpoint_id);
```

## Behavioural Events & Funnels

```sql
CREATE TABLE behavioural_events (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    customer_id     UUID REFERENCES customers(id),
    anonymous_id    TEXT,
    session_id      UUID REFERENCES sessions(id),
    event_name      TEXT NOT NULL CHECK (event_name IN (
        'page_viewed', 'product_viewed', 'product_list_viewed',
        'product_added', 'product_removed', 'cart_viewed',
        'checkout_started', 'checkout_step_viewed', 'checkout_step_completed',
        'payment_info_entered', 'order_completed', 'order_refunded',
        'coupon_entered', 'coupon_applied', 'coupon_denied',
        'promotion_viewed', 'promotion_clicked',
        'product_searched', 'product_shared', 'product_reviewed'
    )),
    occurred_at     TIMESTAMPTZ NOT NULL,
    product_id      UUID REFERENCES products(id),
    properties      JSONB NOT NULL DEFAULT '{}',
    source          TEXT NOT NULL CHECK (source IN ('pixel', 'server_side', 'segment', 'rudderstack', 'ga4', 'custom')),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (occurred_at);

CREATE INDEX idx_behaviour_tenant ON behavioural_events(tenant_id, occurred_at);
CREATE INDEX idx_behaviour_customer ON behavioural_events(customer_id);
CREATE INDEX idx_behaviour_session ON behavioural_events(session_id);
CREATE INDEX idx_behaviour_event ON behavioural_events(tenant_id, event_name);
CREATE INDEX idx_behaviour_product ON behavioural_events(product_id);
```

## AI & Audit

```sql
CREATE TABLE ai_suggestions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    suggestion_type TEXT NOT NULL CHECK (suggestion_type IN (
        'roas_anomaly', 'creative_underperforming', 'stockout_risk',
        'demand_forecast', 'churn_risk', 'expansion_opportunity',
        'attribution_insight', 'funnel_bottleneck', 'campaign_pause',
        'product_recommendation'
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
| Connectors & Channels | 2 | data_sources, channels |
| Products & Inventory | 2 | products, inventory_snapshots |
| Customers & Sessions | 2 | customers, sessions (partitioned) |
| Orders | 2 | orders (partitioned), order_items |
| Attribution | 2 | touchpoints (partitioned), attribution_results |
| Behavioural Events | 1 | behavioural_events (partitioned) |
| AI & Audit | 2 | ai_suggestions, audit_log (partitioned) |
| **Total** | **15** | 5 partitioned tables |

---

## Key Design Decisions

1. **Touchpoints and attribution results separated** — touchpoints store raw interactions (ad clicks, email opens); attribution_results store the computed credit per model per order. This allows running multiple attribution models simultaneously and comparing results without re-ingesting touchpoint data.

2. **Behavioural events follow Segment E-Commerce Spec v2** — event_name values map directly to Segment's canonical names, making ingestion from Segment/Rudderstack a simple category mapping rather than a complex transformation.

3. **Inventory snapshots as daily time series** — inventory_snapshots records daily stock levels per product, enabling sell-through rate and days-of-stock calculations as simple window functions. The is_stockout flag enables alerts tied to active campaign spend.

4. **Orders partitioned by order_date** — high-volume merchants generate millions of orders; date partitioning keeps GMV, AOV, and conversion queries fast with partition pruning.

5. **Sessions link anonymous browsing to identified purchases** — sessions carry both anonymous_id and customer_id, enabling session stitching when a visitor later identifies themselves. The converted flag and order_id provide the conversion anchor.

6. **Multi-currency with base conversion** — orders store both original currency/amount and base currency conversion, with the FX rate preserved for audit. Revenue reports consistently use total_base_cents.

7. **COGS on order items** — cost_cents and margin_cents on order_items enable true profitability (contribution margin) reporting per product, per order, per channel — a key differentiator over tools that stop at revenue.

8. **Creative-level tracking on touchpoints** — creative_id and creative_name on touchpoints feed the creative performance dashboard, linking specific ad assets to conversion outcomes via attribution_results.

9. **Consent-aware customers** — consent_status tracks GDPR/CCPA opt-in state per customer, enabling consent-filtered analytics that only include opted-in users in behavioural reports.

10. **Channel as a first-class entity** — channels are normalised with a channel_type taxonomy, enabling blended ROAS and CAC-by-channel queries across paid and organic sources with consistent grouping.
