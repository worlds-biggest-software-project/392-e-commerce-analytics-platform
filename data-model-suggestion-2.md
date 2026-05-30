# Data Model Suggestion 2: Hybrid Relational + JSONB

> Project: E-Commerce Analytics Platform · Created: 2026-05-29

## Philosophy

This model keeps the high-volume analytical tables relational (orders, sessions, behavioural events, touchpoints) while embedding configuration, product catalogues, channel definitions, and inventory state into JSONB columns on their parent records. The tenant becomes a self-contained configuration document holding data source connections, channel taxonomy, and attribution model settings; products embed their inventory history; and customers embed their order summaries and segment assignments.

E-commerce analytics has a clear split between high-frequency event data (millions of page views, sessions, and touchpoints per day) and relatively stable reference data (product catalogue, channel definitions, inventory levels). The hybrid approach optimises for this: event tables get relational indexes for fast time-range and funnel queries, while reference data benefits from schema-free flexibility as new commerce platforms and ad channels are added without ALTER TABLE migrations.

**Best for:** Teams building an MVP that needs to iterate quickly on product catalogue structures, channel definitions, and attribution model configurations while maintaining relational performance for the hot analytical path (sessions, orders, events).

**Trade-offs:**
- (+) Fewer tables — simpler schema, faster onboarding
- (+) Product catalogue and channel config as JSONB allows rapid iteration without migrations
- (+) Customer record is a complete profile document — single fetch for customer drill-down
- (+) Event and order tables remain fully relational for fast analytical queries
- (-) Product-level queries across all tenants require JSONB operators
- (-) Referential integrity between embedded product IDs and orders validated in application code
- (-) Larger row sizes for tenants with large product catalogues

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| Segment E-Commerce Spec v2 | Behavioural event names align with Segment's canonical e-commerce taxonomy |
| GA4 Measurement Protocol | Event ingestion normalises GA4 event types into canonical event categories |
| ISO 4217 | Currency codes in orders and tenant base_currency |
| ISO 3166-1 | Country codes on customers and sessions |
| OAuth 2.0 | Billing source OAuth credentials referenced in tenant data_sources_json |
| PCI DSS v4.0 | No raw card data; tokenised payment references only |
| GDPR / CCPA | Consent-aware event capture; customer anonymisation preserves aggregates |

---

## Core Tables

```sql
CREATE TABLE tenants (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL UNIQUE,
    base_currency   CHAR(3) NOT NULL DEFAULT 'USD',
    timezone        TEXT NOT NULL DEFAULT 'UTC',

    -- Embedded users
    users_json      JSONB NOT NULL DEFAULT '[]',
    -- [{
    --   "id": "uuid", "email": "...", "name": "...",
    --   "role": "owner|admin|analyst|viewer|api_service",
    --   "is_active": true
    -- }]

    -- Data source connections
    data_sources_json JSONB NOT NULL DEFAULT '[]',
    -- [{
    --   "id": "uuid", "name": "Shopify Production",
    --   "source_type": "shopify|woocommerce|bigcommerce|amazon_seller|meta_ads|google_ads|tiktok_ads|...",
    --   "category": "commerce|advertising|analytics|marketing",
    --   "status": "active", "credentials_ref": "vault://...",
    --   "last_sync_at": "2026-05-29T...", "sync_cursor": "...",
    --   "config": {"sync_interval_minutes": 15}
    -- }]

    -- Channel taxonomy
    channels_json   JSONB NOT NULL DEFAULT '[]',
    -- [{
    --   "id": "uuid", "name": "Meta Paid Social",
    --   "channel_type": "paid_social|paid_search|organic_search|email|sms|direct|referral|affiliate|marketplace",
    --   "data_source_id": "uuid", "is_active": true
    -- }]

    -- Attribution model configuration
    attribution_json JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "models": ["last_click", "first_click", "linear", "probabilistic"],
    --   "default_model": "last_click",
    --   "lookback_window_days": 30,
    --   "probabilistic_config": {
    --     "min_touchpoints": 100, "confidence_threshold": 0.7
    --   }
    -- }

    -- Platform configuration
    config_json     JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "plan": "growth",
    --   "pixel": {"enabled": true, "domain": "track.example.com", "consent_required": true},
    --   "webhooks": [{"url": "...", "events": ["order.*"], "secret_ref": "vault://..."}],
    --   "alerts": {
    --     "stockout_campaign_pause": true,
    --     "roas_threshold": 2.0,
    --     "channels": ["email", "slack"]
    --   },
    --   "export": {"warehouse": "snowflake", "schedule": "daily"}
    -- }

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_tenants_slug ON tenants(slug);
```

## Products & Customers

```sql
CREATE TABLE products (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    data_source_id  UUID NOT NULL,
    external_id     TEXT NOT NULL,
    sku             TEXT,
    name            TEXT NOT NULL,
    category        TEXT,
    brand           TEXT,
    price_cents     BIGINT NOT NULL,
    cost_cents      BIGINT,
    currency        CHAR(3) NOT NULL,
    status          TEXT NOT NULL DEFAULT 'active' CHECK (status IN ('active', 'archived', 'out_of_stock')),
    tags            TEXT[] NOT NULL DEFAULT '{}',

    -- Embedded inventory state
    inventory_json  JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "current": {
    --     "quantity_on_hand": 150, "quantity_reserved": 12, "quantity_available": 138,
    --     "reorder_point": 50, "days_of_stock": 23.5, "is_stockout": false
    --   },
    --   "history": [
    --     {"date": "2026-05-28", "on_hand": 162, "sold": 14, "sell_through_rate": 0.086},
    --     {"date": "2026-05-27", "on_hand": 176, "sold": 18, "sell_through_rate": 0.102}
    --   ],
    --   "forecast": {
    --     "daily_demand_avg": 15, "stockout_date_est": "2026-06-08",
    --     "reorder_suggested": true, "reorder_quantity": 200
    --   }
    -- }

    -- Performance metrics (pre-computed)
    metrics_json    JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "total_revenue_cents": 4950000, "total_units_sold": 500,
    --   "total_orders": 420, "avg_order_qty": 1.19,
    --   "margin_cents": 1980000, "margin_pct": 0.40,
    --   "views": 12500, "add_to_cart": 1800, "view_to_cart_rate": 0.144,
    --   "cart_to_purchase_rate": 0.233,
    --   "return_rate": 0.05, "avg_rating": 4.2
    -- }

    attributes      JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, data_source_id, external_id)
);

CREATE INDEX idx_products_tenant ON products(tenant_id);
CREATE INDEX idx_products_sku ON products(tenant_id, sku);
CREATE INDEX idx_products_category ON products(tenant_id, category);
CREATE INDEX idx_products_tags ON products USING GIN (tags);
CREATE INDEX idx_products_inventory ON products USING GIN (inventory_json);

CREATE TABLE customers (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    data_source_id  UUID NOT NULL,
    external_id     TEXT NOT NULL,
    email           TEXT,
    name            TEXT,
    country         CHAR(2),
    first_seen_at   TIMESTAMPTZ NOT NULL,
    cohort_month    DATE,
    segment         TEXT CHECK (segment IN ('new', 'active', 'high_ltv', 'at_risk', 'lapsed', 'churned')),
    acquisition_channel TEXT,
    consent_status  TEXT NOT NULL DEFAULT 'unknown' CHECK (consent_status IN ('opted_in', 'opted_out', 'unknown')),
    tags            TEXT[] NOT NULL DEFAULT '{}',

    -- Embedded order summary
    orders_json     JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "total_orders": 8, "total_revenue_cents": 125000,
    --   "first_order_at": "2025-03-15T...", "last_order_at": "2026-05-20T...",
    --   "aov_cents": 15625, "ltv_cents": 125000,
    --   "is_repeat": true, "repeat_purchase_count": 7,
    --   "avg_days_between_orders": 52,
    --   "top_categories": ["electronics", "accessories"],
    --   "top_products": [{"id": "uuid", "name": "Widget Pro", "orders": 3}],
    --   "refund_count": 0, "refund_total_cents": 0
    -- }

    -- Attribution summary
    attribution_json JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "first_touch_channel": "paid_social",
    --   "first_touch_campaign": "spring_sale_2025",
    --   "touchpoint_count": 12,
    --   "channels_touched": ["paid_social", "email", "direct"],
    --   "days_to_first_purchase": 7
    -- }

    -- Behavioural summary
    behaviour_json  JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "last_active_at": "2026-05-28T...",
    --   "sessions_30d": 5, "page_views_30d": 42,
    --   "products_viewed_30d": 15, "cart_adds_30d": 3,
    --   "churn_risk_score": 0.15,
    --   "predicted_next_order_date": "2026-06-15"
    -- }

    custom_attributes JSONB NOT NULL DEFAULT '{}',
    is_deleted      BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, data_source_id, external_id)
);

CREATE INDEX idx_customers_tenant ON customers(tenant_id);
CREATE INDEX idx_customers_segment ON customers(tenant_id, segment);
CREATE INDEX idx_customers_cohort ON customers(tenant_id, cohort_month);
CREATE INDEX idx_customers_tags ON customers USING GIN (tags);
```

## Orders & Events

```sql
CREATE TABLE orders (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    customer_id     UUID NOT NULL REFERENCES customers(id),
    data_source_id  UUID NOT NULL,
    external_id     TEXT NOT NULL,
    order_date      TIMESTAMPTZ NOT NULL,
    status          TEXT NOT NULL CHECK (status IN (
        'pending', 'confirmed', 'fulfilled', 'shipped', 'delivered',
        'cancelled', 'refunded', 'partially_refunded'
    )),
    total_cents     BIGINT NOT NULL,
    subtotal_cents  BIGINT NOT NULL,
    discount_cents  BIGINT NOT NULL DEFAULT 0,
    shipping_cents  BIGINT NOT NULL DEFAULT 0,
    tax_cents       BIGINT NOT NULL DEFAULT 0,
    currency        CHAR(3) NOT NULL,
    total_base_cents BIGINT NOT NULL,
    fx_rate         NUMERIC(18,8) NOT NULL DEFAULT 1.0,
    is_first_order  BOOLEAN NOT NULL DEFAULT false,
    coupon_code     TEXT,
    shipping_country CHAR(2),

    -- Embedded line items
    items_json      JSONB NOT NULL DEFAULT '[]',
    -- [{
    --   "product_id": "uuid", "sku": "WDG-PRO-001", "name": "Widget Pro",
    --   "quantity": 2, "unit_price_cents": 4900, "discount_cents": 0,
    --   "total_cents": 9800, "cost_cents": 1960, "margin_cents": 7840,
    --   "category": "electronics"
    -- }]

    -- Attribution snapshot (computed at order time)
    attribution_json JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "session_id": "uuid", "channel": "paid_social", "channel_id": "uuid",
    --   "campaign": "summer_2026", "creative_id": "cr_123",
    --   "touchpoints": [
    --     {"id": "uuid", "channel": "paid_social", "at": "2026-05-20T...",
    --      "models": {"last_click": 0.0, "first_click": 1.0, "linear": 0.33}},
    --     {"id": "uuid", "channel": "email", "at": "2026-05-23T...",
    --      "models": {"last_click": 0.0, "first_click": 0.0, "linear": 0.33}},
    --     {"id": "uuid", "channel": "direct", "at": "2026-05-25T...",
    --      "models": {"last_click": 1.0, "first_click": 0.0, "linear": 0.33}}
    --   ],
    --   "days_to_conversion": 5
    -- }

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, data_source_id, external_id)
) PARTITION BY RANGE (order_date);

CREATE INDEX idx_orders_tenant_date ON orders(tenant_id, order_date);
CREATE INDEX idx_orders_customer ON orders(customer_id);
CREATE INDEX idx_orders_status ON orders(tenant_id, status);
CREATE INDEX idx_orders_attribution ON orders USING GIN (attribution_json);

CREATE TABLE behavioural_events (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    customer_id     UUID REFERENCES customers(id),
    anonymous_id    TEXT,
    session_id      TEXT,
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
    source          TEXT NOT NULL CHECK (source IN ('pixel', 'server_side', 'segment', 'rudderstack', 'ga4', 'custom')),

    -- Event-specific properties
    properties_json JSONB NOT NULL DEFAULT '{}',
    -- For product_viewed: {"product_name": "...", "category": "...", "price_cents": 4900}
    -- For checkout_step_viewed: {"step": 2, "step_name": "shipping", "payment_method": "credit_card"}
    -- For product_searched: {"query": "wireless headphones", "results_count": 42}

    -- Session context
    context_json    JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "device_type": "mobile", "browser": "Chrome", "os": "iOS",
    --   "country": "US", "city": "New York",
    --   "utm_source": "meta", "utm_medium": "cpc", "utm_campaign": "summer_2026",
    --   "referrer": "https://facebook.com", "landing_page": "/products/widget-pro"
    -- }

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (occurred_at);

CREATE INDEX idx_behaviour_tenant ON behavioural_events(tenant_id, occurred_at);
CREATE INDEX idx_behaviour_customer ON behavioural_events(customer_id);
CREATE INDEX idx_behaviour_event ON behavioural_events(tenant_id, event_name);
CREATE INDEX idx_behaviour_product ON behavioural_events(product_id);
CREATE INDEX idx_behaviour_session ON behavioural_events(tenant_id, session_id);
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
| Configuration | 1 | tenants (embeds users, data sources, channels, attribution config) |
| Products & Customers | 2 | products (embeds inventory, metrics), customers (embeds orders, attribution, behaviour) |
| Orders & Events | 2 | orders (partitioned, embeds items and attribution), behavioural_events (partitioned) |
| AI & Audit | 2 | ai_suggestions, audit_log (partitioned) |
| **Total** | **7** | 3 partitioned tables |

---

## Key Design Decisions

1. **Attribution embedded on orders** — each order carries its full touchpoint chain with per-model credit weights in attribution_json. ROAS-by-channel queries use `attribution_json->>'channel'` without joining a separate attribution table. The touchpoint array preserves the journey for drill-down.

2. **Line items embedded on orders** — items_json collocates product, quantity, price, cost, and margin on the order row. Margin analysis is a single-table query. Products are referenced by ID for cross-order product analytics.

3. **Product inventory and metrics embedded** — products.inventory_json tracks current stock, daily history, and demand forecast in one document. The dashboard can show sell-through, days-of-stock, and stockout risk from a single product fetch. products.metrics_json pre-computes revenue, view-to-cart rate, and return rate.

4. **Customer as a complete profile** — customers embed order summary (total orders, LTV, AOV, repeat purchase patterns), attribution (first touch, channels touched), and behavioural summary (recent activity, churn risk). The customer drill-down view is a single-row read.

5. **Channel taxonomy on tenant** — channels_json embeds channel definitions (paid_social, email, direct, etc.) on the tenant, avoiding a separate table. Orders reference channel by ID via attribution_json.

6. **Behavioural events remain relational** — the highest-volume table stays fully relational with partitioning for fast funnel and cohort queries. Event names follow Segment E-Commerce Spec v2 as CHECK constraints.

7. **Session context embedded on events** — instead of a separate sessions table, context_json on behavioural_events carries device, geo, UTM, and referrer data. Session-level aggregation uses GROUP BY session_id.

8. **Stockout-to-campaign integration** — products.inventory_json.current.is_stockout combined with tenants.config_json.alerts.stockout_campaign_pause enables the key differentiator: automatically flagging out-of-stock products that have active ad spend.
