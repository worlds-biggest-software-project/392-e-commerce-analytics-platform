# Data Model Suggestion 3: Event-Sourced / Audit-First

> Project: E-Commerce Analytics Platform · Created: 2026-05-29

## Philosophy

This model treats every shopper interaction, order, inventory change, and attribution signal as an immutable event in a single append-only store. The conversion funnel is not a report — it is a projection replayed from page_viewed → product_viewed → product_added → checkout_started → order_completed event sequences. Attribution is computed by replaying touchpoint events through configurable models. Inventory levels are derived from stock_received and stock_sold events rather than maintained as mutable quantities.

E-commerce data is inherently event-driven: commerce platforms (Shopify, BigCommerce) and analytics tools (Segment, GA4) already emit event streams following well-defined taxonomies. An event-sourced model preserves the raw temporal sequence, enabling retroactive re-attribution (e.g., switching from last-click to probabilistic without data loss), "as-of" reporting for investor updates, and AI churn-risk models trained on complete behavioural sequences rather than pre-aggregated summaries.

**Best for:** Platforms requiring full behavioural sequence preservation for AI/ML feature engineering, retroactive re-attribution across models, temporal queries ("what was GMV on date X?"), and regulatory compliance requiring traceable data lineage from raw event to reported metric.

**Trade-offs:**
- (+) Complete behavioural sequences preserved for AI training and cohort survival analysis
- (+) Re-attribution without re-ingestion: replay touchpoint events through a new model
- (+) "As-of" queries at any historical point without maintaining snapshot tables
- (+) New metrics added by creating new projections rather than backfilling columns
- (-) Read models must be maintained and rebuilt when projection logic changes
- (-) Simple lookups ("current GMV today") require read models, not direct queries
- (-) Event replay for high-traffic stores can be slow without snapshots
- (-) Higher storage requirements for full event history

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| Segment E-Commerce Spec v2 | Event types map directly to Segment's canonical e-commerce taxonomy (product_viewed, order_completed, etc.) |
| GA4 Measurement Protocol | GA4 events normalised into canonical event types at ingestion |
| CloudEvents v1.0 | Event store uses CloudEvents envelope (ce_source, ce_type, ce_time) for vendor-neutral event format |
| AsyncAPI 3.x | Event taxonomy documented as AsyncAPI channels for external consumers |
| ISO 4217 | Currency codes on order and revenue events |
| ISO 3166-1 | Country codes on customer and session events |
| GDPR / CCPA | Crypto-shredding: customer PII encrypted per-customer; key deletion renders events unreadable while preserving aggregates |

---

## Event Infrastructure

```sql
CREATE TABLE event_store (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_type     TEXT NOT NULL CHECK (stream_type IN (
        'tenant', 'user', 'data_source', 'channel',
        'product', 'inventory', 'customer', 'session',
        'order', 'touchpoint', 'attribution',
        'campaign', 'creative', 'ai', 'config'
    )),
    stream_id       UUID NOT NULL,
    version         BIGINT NOT NULL,
    event_type      TEXT NOT NULL,
    actor_type      TEXT NOT NULL CHECK (actor_type IN (
        'user', 'system', 'api_key', 'webhook', 'ai',
        'pixel', 'server_side', 'commerce_platform',
        'ad_platform', 'scheduler', 'projection_engine'
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
- `tenant_created` — {name, slug, base_currency, timezone, plan}
- `tenant_config_updated` — {field, old_value, new_value}
- `user_added` — {user_id, email, role}
- `data_source_connected` — {source_type, category, config}
- `data_source_sync_completed` — {events_ingested, duration_ms}
- `channel_defined` — {name, channel_type, data_source_id}
- `attribution_model_configured` — {models, default_model, lookback_window_days}

### Product Events
- `product_imported` — {external_id, sku, name, category, brand, price_cents, cost_cents, currency}
- `product_updated` — {fields_changed}
- `product_price_changed` — {old_price_cents, new_price_cents}
- `product_archived` — {}
- `product_tagged` — {tags_added, tags_removed}

### Inventory Events
- `stock_received` — {product_id, quantity, source: purchase_order|return|adjustment}
- `stock_sold` — {product_id, quantity, order_id}
- `stock_reserved` — {product_id, quantity, order_id}
- `stock_released` — {product_id, quantity, reason: cancel|expiry}
- `stock_adjusted` — {product_id, quantity_delta, reason}
- `stockout_detected` — {product_id, active_campaigns: [{campaign_id, daily_spend_cents}]}
- `reorder_point_reached` — {product_id, quantity_on_hand, reorder_point}

### Customer Events
- `customer_created` — {external_id, email, name, country, acquisition_channel}
- `customer_identified` — {anonymous_id, customer_id} (session stitching)
- `customer_updated` — {fields_changed}
- `customer_segment_changed` — {old_segment, new_segment}
- `customer_tagged` — {tags_added, tags_removed}
- `customer_consent_updated` — {consent_status, jurisdiction}
- `customer_deleted` — {gdpr_request: true}

### Session Events
- `session_started` — {anonymous_id, customer_id, device_type, browser, os, country, landing_page, utm_source, utm_medium, utm_campaign, referrer}
- `session_identified` — {anonymous_id, customer_id}
- `session_ended` — {duration_seconds, page_views, converted, order_id}

### Behavioural Events (Segment E-Commerce Spec v2)
- `page_viewed` — {url, title, referrer}
- `product_viewed` — {product_id, sku, name, category, price_cents}
- `product_list_viewed` — {list_id, category, products: [{product_id, position}]}
- `product_added` — {product_id, quantity, price_cents, cart_id}
- `product_removed` — {product_id, quantity, cart_id}
- `cart_viewed` — {cart_id, products: [{product_id, quantity}], total_cents}
- `checkout_started` — {cart_id, total_cents, item_count}
- `checkout_step_viewed` — {step, step_name}
- `checkout_step_completed` — {step, step_name, payment_method}
- `payment_info_entered` — {payment_method}
- `product_searched` — {query, results_count}
- `promotion_viewed` — {promotion_id, name, creative}
- `promotion_clicked` — {promotion_id, name}

### Order Events
- `order_placed` — {external_id, customer_id, items: [{product_id, quantity, unit_price_cents, cost_cents}], subtotal_cents, discount_cents, shipping_cents, tax_cents, total_cents, currency, fx_rate, total_base_cents, coupon_code, shipping_country, is_first_order, session_id}
- `order_confirmed` — {}
- `order_fulfilled` — {tracking_number}
- `order_shipped` — {carrier, estimated_delivery}
- `order_delivered` — {}
- `order_cancelled` — {reason}
- `order_refunded` — {refund_amount_cents, items: [{product_id, quantity}]}
- `order_partially_refunded` — {refund_amount_cents, items: [{product_id, quantity}]}

### Touchpoint Events
- `touchpoint_recorded` — {customer_id, anonymous_id, channel_id, touchpoint_type, campaign_id, campaign_name, ad_group_id, creative_id, creative_name, keyword, landing_page, cost_cents, impressions, clicks}

### Attribution Events
- `attribution_computed` — {order_id, model, touchpoints: [{touchpoint_id, channel_id, credit_weight, attributed_revenue_cents}]}
- `attribution_recomputed` — {order_id, model, reason, touchpoints: [...]}

### Campaign & Creative Events
- `campaign_spend_recorded` — {channel_id, campaign_id, date, spend_cents, impressions, clicks, currency}
- `creative_performance_recorded` — {creative_id, campaign_id, channel_id, date, spend_cents, impressions, clicks, conversions, revenue_cents}
- `campaign_paused_for_stockout` — {campaign_id, product_id, reason}

### AI Events
- `ai_suggestion_generated` — {suggestion_type, entity_type, entity_id, title, description, evidence, confidence}
- `ai_suggestion_accepted` — {suggestion_id}
- `ai_suggestion_dismissed` — {suggestion_id, reason}
- `ai_churn_risk_scored` — {customer_id, risk_score, risk_factors, model_version}
- `ai_demand_forecast_generated` — {product_id, forecast: [{date, predicted_demand, confidence_lower, confidence_upper}]}
- `ai_roas_anomaly_detected` — {channel_id, campaign_id, expected_roas, actual_roas, deviation_pct}
- `ai_creative_analysis_completed` — {creative_id, performance_tier, recommendation}

---

## Read Models

```sql
CREATE TABLE rm_store_dashboard (
    tenant_id       UUID NOT NULL,
    dashboard_date  DATE NOT NULL,

    -- Revenue metrics
    gmv_cents           BIGINT NOT NULL DEFAULT 0,
    gmv_base_cents      BIGINT NOT NULL DEFAULT 0,
    total_orders        INT NOT NULL DEFAULT 0,
    aov_cents           BIGINT,
    total_customers     INT NOT NULL DEFAULT 0,
    new_customers       INT NOT NULL DEFAULT 0,
    returning_customers INT NOT NULL DEFAULT 0,

    -- Conversion funnel
    sessions            INT NOT NULL DEFAULT 0,
    product_views       INT NOT NULL DEFAULT 0,
    add_to_carts        INT NOT NULL DEFAULT 0,
    checkouts_started   INT NOT NULL DEFAULT 0,
    orders_completed    INT NOT NULL DEFAULT 0,
    conversion_rate     NUMERIC(8,6),

    -- Channel breakdown
    channel_breakdown   JSONB NOT NULL DEFAULT '[]',
    -- [{
    --   "channel_id": "uuid", "channel_name": "Meta Paid Social",
    --   "sessions": 5000, "orders": 120, "revenue_cents": 1200000,
    --   "spend_cents": 350000, "roas": 3.43, "cac_cents": 2917,
    --   "attributed_revenue": {
    --     "last_click": 1200000, "first_click": 980000,
    --     "linear": 1050000, "probabilistic": 1100000
    --   }
    -- }]

    -- Device breakdown
    device_breakdown    JSONB NOT NULL DEFAULT '{}',
    -- {"desktop": {"sessions": 3000, "orders": 80}, "mobile": {"sessions": 6000, "orders": 40}}

    -- Category breakdown
    category_breakdown  JSONB NOT NULL DEFAULT '[]',

    last_event_id   UUID NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (tenant_id, dashboard_date)
) PARTITION BY RANGE (dashboard_date);

CREATE TABLE rm_cohort_retention (
    tenant_id       UUID NOT NULL,
    cohort_month    DATE NOT NULL,
    acquisition_channel TEXT,

    initial_customers   INT NOT NULL,
    initial_revenue_cents BIGINT NOT NULL,

    -- Period-by-period retention
    periods         JSONB NOT NULL DEFAULT '[]',
    -- [{
    --   "period": 0, "month": "2025-06-01",
    --   "customers": 200, "orders": 200, "revenue_cents": 3000000,
    --   "repeat_rate": 1.0, "revenue_retention": 1.0
    -- }, {
    --   "period": 1, "month": "2025-07-01",
    --   "customers": 60, "orders": 72, "revenue_cents": 1080000,
    --   "repeat_rate": 0.30, "revenue_retention": 0.36
    -- }]

    last_event_id   UUID NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (tenant_id, cohort_month, acquisition_channel)
);

CREATE TABLE rm_product_performance (
    tenant_id       UUID NOT NULL,
    product_id      UUID NOT NULL,

    -- Product identity
    sku             TEXT,
    name            TEXT NOT NULL,
    category        TEXT,
    brand           TEXT,
    price_cents     BIGINT NOT NULL,
    cost_cents      BIGINT,

    -- Sales metrics
    total_revenue_cents BIGINT NOT NULL DEFAULT 0,
    total_units_sold    INT NOT NULL DEFAULT 0,
    total_orders        INT NOT NULL DEFAULT 0,
    margin_cents        BIGINT NOT NULL DEFAULT 0,
    margin_pct          NUMERIC(8,4),
    return_count        INT NOT NULL DEFAULT 0,
    return_rate         NUMERIC(8,6),

    -- Behavioural metrics
    views               INT NOT NULL DEFAULT 0,
    add_to_carts        INT NOT NULL DEFAULT 0,
    view_to_cart_rate   NUMERIC(8,6),
    cart_to_purchase_rate NUMERIC(8,6),

    -- Inventory state (from inventory events)
    quantity_on_hand    INT,
    quantity_available  INT,
    days_of_stock       NUMERIC(8,2),
    sell_through_rate   NUMERIC(8,4),
    is_stockout         BOOLEAN NOT NULL DEFAULT false,
    stockout_since      TIMESTAMPTZ,
    active_campaign_spend_cents BIGINT NOT NULL DEFAULT 0,

    -- Demand forecast
    forecast_json   JSONB,
    -- {
    --   "daily_demand_avg": 15,
    --   "stockout_date_est": "2026-06-08",
    --   "reorder_suggested": true, "reorder_quantity": 200,
    --   "forecast": [{"date": "2026-06-01", "predicted": 14, "lower": 8, "upper": 20}]
    -- }

    last_event_id   UUID NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (tenant_id, product_id)
);

CREATE INDEX idx_rm_product_category ON rm_product_performance(tenant_id, category);
CREATE INDEX idx_rm_product_stockout ON rm_product_performance(tenant_id) WHERE is_stockout;

CREATE TABLE rm_channel_performance (
    tenant_id       UUID NOT NULL,
    channel_id      UUID NOT NULL,
    period_date     DATE NOT NULL,

    -- Spend & volume
    spend_cents         BIGINT NOT NULL DEFAULT 0,
    impressions         BIGINT NOT NULL DEFAULT 0,
    clicks              BIGINT NOT NULL DEFAULT 0,
    sessions            INT NOT NULL DEFAULT 0,

    -- Attribution by model
    attribution_json    JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "last_click": {"orders": 120, "revenue_cents": 1200000, "roas": 3.43},
    --   "first_click": {"orders": 95, "revenue_cents": 980000, "roas": 2.80},
    --   "linear": {"orders": 108, "revenue_cents": 1050000, "roas": 3.00},
    --   "probabilistic": {"orders": 112, "revenue_cents": 1100000, "roas": 3.14}
    -- }

    -- Blended metrics
    blended_roas        NUMERIC(10,4),
    cac_cents           BIGINT,
    cpc_cents           BIGINT,
    ctr                 NUMERIC(8,6),

    -- Creative breakdown
    creative_breakdown  JSONB NOT NULL DEFAULT '[]',
    -- [{
    --   "creative_id": "cr_123", "creative_name": "Summer Banner A",
    --   "spend_cents": 50000, "impressions": 25000, "clicks": 750,
    --   "conversions": 30, "revenue_cents": 300000, "roas": 6.0
    -- }]

    last_event_id   UUID NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (tenant_id, channel_id, period_date)
);

CREATE TABLE rm_customer_journey (
    tenant_id       UUID NOT NULL,
    customer_id     UUID NOT NULL,

    -- Identity
    external_id     TEXT NOT NULL,
    email           TEXT,
    name            TEXT,
    country         CHAR(2),
    segment         TEXT,
    cohort_month    DATE,
    consent_status  TEXT,

    -- Lifetime metrics
    total_orders        INT NOT NULL DEFAULT 0,
    total_revenue_cents BIGINT NOT NULL DEFAULT 0,
    ltv_cents           BIGINT NOT NULL DEFAULT 0,
    aov_cents           BIGINT,
    first_order_at      TIMESTAMPTZ,
    last_order_at       TIMESTAMPTZ,

    -- Acquisition
    acquisition_channel TEXT,
    first_touch_campaign TEXT,
    days_to_first_purchase INT,

    -- Behavioural state
    last_active_at      TIMESTAMPTZ,
    sessions_30d        INT NOT NULL DEFAULT 0,
    churn_risk_score    NUMERIC(5,4),
    predicted_next_order DATE,

    -- Recent journey (most recent N events)
    journey         JSONB NOT NULL DEFAULT '[]',
    -- [{
    --   "at": "2026-05-28T10:15:00Z", "type": "product_viewed",
    --   "product": "Widget Pro", "channel": "organic_search"
    -- }, {
    --   "at": "2026-05-28T10:18:00Z", "type": "product_added",
    --   "product": "Widget Pro"
    -- }, {
    --   "at": "2026-05-28T10:25:00Z", "type": "order_placed",
    --   "total_cents": 4900, "is_first_order": false
    -- }]

    last_event_id   UUID NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (tenant_id, customer_id)
);

CREATE INDEX idx_rm_customer_segment ON rm_customer_journey(tenant_id, segment);
CREATE INDEX idx_rm_customer_churn ON rm_customer_journey(tenant_id, churn_risk_score DESC NULLS LAST);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Infrastructure | 3 | event_store (partitioned), stream_snapshots, projection_checkpoints |
| Read Models | 5 | rm_store_dashboard (partitioned), rm_cohort_retention, rm_product_performance, rm_channel_performance, rm_customer_journey |
| **Total** | **8** | 2 partitioned tables |

---

## Key Design Decisions

1. **Funnel as event replay** — the conversion funnel (page → product view → cart → checkout → order) is computed by replaying session and behavioural events in sequence. Adding a new funnel step (e.g., "wishlist_added") means adding a new event type and updating the projection — no schema migration.

2. **Attribution as replayable events** — touchpoint_recorded events capture raw interactions; attribution_computed events store model results. Re-attribution with a different model replays the touchpoints and emits new attribution_computed events without touching the originals.

3. **Inventory as events, not mutable state** — stock_received, stock_sold, stock_adjusted events drive the rm_product_performance projection. Inventory discrepancies are traceable to specific events. The stockout_detected event enables the campaign-pause integration.

4. **Channel performance with multi-model attribution** — rm_channel_performance stores per-model attribution results (last_click, first_click, linear, probabilistic) side by side, enabling the ROAS comparison dashboard without re-querying the event store.

5. **Creative-level performance embedded in channel read model** — creative_breakdown on rm_channel_performance avoids a separate creative table while still supporting the creative analytics dashboard (ranking ads by ROAS and engagement).

6. **Customer journey as a read model** — rm_customer_journey materialises the full customer profile including lifetime metrics, acquisition context, behavioural state, churn risk, and recent event timeline. All derived from events; no mutable customer "master record."

7. **Crypto-shredding for GDPR compliance** — customer PII in events is encrypted with a per-customer key. customer_deleted event triggers key destruction, making all historical events for that customer unreadable while preserving aggregate projections (GMV totals, cohort counts).

8. **Segment E-Commerce Spec v2 as native event taxonomy** — behavioural event types map directly to Segment's canonical names, making ingestion from Segment/Rudderstack/GA4 a simple mapping rather than a complex transformation.

9. **Campaign and creative as separate event streams** — campaign_spend_recorded and creative_performance_recorded events are ingested from ad platforms independently of behavioural events, enabling spend-to-revenue correlation in the channel performance projection.

10. **AI events for closed-loop optimisation** — ai_demand_forecast_generated, ai_roas_anomaly_detected, and ai_creative_analysis_completed events are first-class citizens in the event store, making AI recommendations auditable and enabling feedback loops (was the forecast accurate? was the creative recommendation followed?).
