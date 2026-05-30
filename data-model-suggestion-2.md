# Data Model Suggestion 2: Hybrid Relational + JSONB

> Project: Retail Store Analytics · Created: 2026-05-29

## Philosophy

This model keeps hourly traffic counts and POS transactions as relational tables (for time-series queries and conversion rate calculations) while embedding sensors, staff schedules, basket rules, context overlays, and store configuration into JSONB on their parent records. Each store is a self-contained operational document with embedded sensors, operating hours, current schedules, and recent basket insights. The tenant holds the user roster, data source connections, and multi-store benchmarking configuration.

Retail store analytics has two access patterns: time-series queries on traffic and transactions ("hourly conversion rate for this store over 30 days") that need relational indexing; and store-level operational queries ("show me everything about this store including current schedule and alerts") that benefit from document-style access.

**Best for:** Teams building a rapid MVP for SMB retailers where store configuration, sensor setup, and scheduling evolve frequently, while maintaining relational performance for traffic time-series and conversion rate dashboards.

**Trade-offs:**
- (+) Store is a complete operational document — single fetch for store manager view
- (+) New sensor types or schedule parameters added as JSONB without ALTER TABLE
- (+) Traffic and transactions remain relational for fast time-series queries
- (-) Cross-store basket analysis requires JSONB operators or application-level aggregation
- (-) Staff schedule changes update embedded JSONB arrays
- (-) Larger store rows for locations with many sensors and complex schedules

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| NRF ARTS ODM | Store and transaction naming conventions |
| EU AI Act | Privacy-by-design: no biometric data stored |
| GDPR | No PII in traffic data |
| OpenAPI 3.1 | REST API |
| OAuth 2.0 | POS platform authentication |

---

## Core Tables

```sql
CREATE TABLE tenants (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL UNIQUE,
    base_currency   CHAR(3) NOT NULL DEFAULT 'USD',

    users_json      JSONB NOT NULL DEFAULT '[]',

    data_sources_json JSONB NOT NULL DEFAULT '[]',
    -- [{"id": "uuid", "name": "Square POS", "source_type": "square|lightspeed|shopify|sensor_api|csv",
    --   "status": "active", "credentials_ref": "vault://..."}]

    config_json     JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "plan": "professional",
    --   "benchmarking": {"enabled": true, "metrics": ["conversion_rate", "aov", "items_per_transaction"]},
    --   "alerts": {"channels": ["slack", "email"], "staffing_threshold": 0.05},
    --   "scheduling": {"forecast_model": "prophet", "min_staff_to_visitor_ratio": 0.05}
    -- }

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_tenants_slug ON tenants(slug);
```

## Stores

```sql
CREATE TABLE stores (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    name            TEXT NOT NULL,
    code            TEXT,
    address         TEXT,
    city            TEXT,
    country         CHAR(2) NOT NULL,
    latitude        NUMERIC(9,6),
    longitude       NUMERIC(9,6),
    timezone        TEXT NOT NULL DEFAULT 'UTC',
    is_active       BOOLEAN NOT NULL DEFAULT true,

    -- Embedded sensors
    sensors_json    JSONB NOT NULL DEFAULT '[]',
    -- [{"id": "uuid", "name": "Front Door Counter", "sensor_type": "people_counter",
    --   "vendor": "dor", "zone": "entrance", "status": "active"}]

    -- Operating hours
    hours_json      JSONB NOT NULL DEFAULT '{}',
    -- {"mon": {"open": "09:00", "close": "21:00"}, ...}

    -- Current staff schedule (this week)
    schedule_json   JSONB NOT NULL DEFAULT '[]',
    -- [{"date": "2026-05-29", "hours": [
    --   {"hour": 9, "predicted_visitors": 45, "recommended_staff": 3, "actual_staff": 3, "service_level": "optimal"},
    --   {"hour": 10, "predicted_visitors": 62, "recommended_staff": 4, "actual_staff": 3, "service_level": "understaffed"}
    -- ]}]

    -- Basket analysis results
    basket_json     JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "analysis_date": "2026-05-25",
    --   "top_rules": [
    --     {"antecedent": ["Coffee"], "consequent": ["Pastry"], "support": 0.15, "confidence": 0.62, "lift": 2.8,
    --      "recommendation": "Place pastry display near coffee counter"}
    --   ],
    --   "basket_metrics": {"avg_items": 3.2, "avg_value_cents": 4500, "avg_basket_size": 2.8}
    -- }

    -- Context overlays
    overlays_json   JSONB NOT NULL DEFAULT '[]',
    -- [{"date": "2026-05-29", "type": "weather", "details": {"temp_c": 22, "condition": "sunny"}},
    --  {"date": "2026-05-30", "type": "promotion", "details": {"name": "Weekend Sale", "discount_pct": 15}}]

    -- Store performance summary (pre-computed)
    performance_json JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "today": {"visitors": 185, "transactions": 42, "conversion_rate": 0.227, "revenue_cents": 189000},
    --   "this_week": {"visitors": 1250, "transactions": 290, "conversion_rate": 0.232, "revenue_cents": 1305000},
    --   "this_month": {"visitors": 5200, "transactions": 1180, "conversion_rate": 0.227, "revenue_cents": 5310000},
    --   "yoy_change": {"visitors_pct": 0.028, "conversion_pct": 0.015}
    -- }

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, name)
);

CREATE INDEX idx_stores_tenant ON stores(tenant_id);
CREATE INDEX idx_stores_geo ON stores(latitude, longitude) WHERE latitude IS NOT NULL;
```

## Traffic & Transactions

```sql
CREATE TABLE traffic_counts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    store_id        UUID NOT NULL REFERENCES stores(id),
    zone            TEXT,
    count_date      DATE NOT NULL,
    count_hour      SMALLINT NOT NULL CHECK (count_hour BETWEEN 0 AND 23),
    entries         INT NOT NULL DEFAULT 0,
    exits           INT NOT NULL DEFAULT 0,
    staff_excluded  INT NOT NULL DEFAULT 0,
    net_visitors    INT NOT NULL,
    peak_occupancy  INT,
    dwell_time_avg_seconds INT,
    data_source     TEXT NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, store_id, zone, count_date, count_hour)
) PARTITION BY RANGE (count_date);

CREATE INDEX idx_traffic_store ON traffic_counts(store_id, count_date);
CREATE INDEX idx_traffic_tenant ON traffic_counts(tenant_id, count_date);

CREATE TABLE pos_transactions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    store_id        UUID NOT NULL REFERENCES stores(id),
    external_id     TEXT NOT NULL,
    transaction_date TIMESTAMPTZ NOT NULL,
    transaction_type TEXT NOT NULL CHECK (transaction_type IN ('sale', 'return', 'exchange', 'void')),
    total_cents     BIGINT NOT NULL,
    currency        CHAR(3) NOT NULL,
    items_count     INT NOT NULL,
    staff_name      TEXT,
    payment_method  TEXT,

    -- Embedded line items
    items_json      JSONB NOT NULL DEFAULT '[]',
    -- [{"sku": "SKU001", "name": "Espresso", "category": "Beverages", "qty": 1, "price_cents": 450},
    --  {"sku": "SKU042", "name": "Croissant", "category": "Pastries", "qty": 1, "price_cents": 350}]

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, store_id, external_id)
) PARTITION BY RANGE (transaction_date);

CREATE INDEX idx_pos_store ON pos_transactions(store_id, transaction_date);
CREATE INDEX idx_pos_tenant ON pos_transactions(tenant_id, transaction_date);
```

## AI & Audit

```sql
CREATE TABLE ai_suggestions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    suggestion_type TEXT NOT NULL CHECK (suggestion_type IN (
        'traffic_anomaly', 'conversion_drop', 'staffing_alert',
        'layout_recommendation', 'promotion_impact', 'weather_impact',
        'queue_alert', 'benchmark_insight', 'schedule_optimization',
        'basket_insight'
    )),
    entity_type     TEXT,
    entity_id       UUID,
    store_id        UUID REFERENCES stores(id),
    severity        TEXT NOT NULL CHECK (severity IN ('info', 'warning', 'critical')),
    title           TEXT NOT NULL,
    description     TEXT NOT NULL,
    evidence        JSONB NOT NULL DEFAULT '{}',
    recommended_action TEXT,
    status          TEXT NOT NULL DEFAULT 'pending' CHECK (status IN ('pending', 'accepted', 'dismissed', 'expired', 'auto_applied')),
    confidence      NUMERIC(5,4),
    model_version   TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    resolved_at     TIMESTAMPTZ
);

CREATE INDEX idx_ai_suggestions_tenant ON ai_suggestions(tenant_id);
CREATE INDEX idx_ai_suggestions_store ON ai_suggestions(store_id);
CREATE INDEX idx_ai_suggestions_pending ON ai_suggestions(tenant_id) WHERE status = 'pending';

CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    actor_type      TEXT NOT NULL CHECK (actor_type IN ('user', 'system', 'api_key', 'sensor', 'ai', 'scheduler')),
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
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Configuration | 1 | tenants (embeds users, data sources, config) |
| Stores | 1 | stores (embeds sensors, hours, schedules, basket rules, overlays, performance) |
| Traffic & Transactions | 2 | traffic_counts (partitioned), pos_transactions (partitioned, embeds items) |
| AI & Audit | 2 | ai_suggestions, audit_log (partitioned) |
| **Total** | **6** | 3 partitioned tables |

---

## Key Design Decisions

1. **Store as a complete operational document** — sensors, operating hours, current schedule, basket analysis results, context overlays, and performance summary all embed on the store. The store manager's dashboard is a single-row fetch.

2. **Traffic and transactions remain relational** — the highest-volume tables stay relational with partitioning for efficient time-series queries: hourly conversion rates, daily comparisons, and period-over-period trends.

3. **Line items embedded on transactions** — pos_transactions.items_json collocates items with the transaction. Basket analysis reads items from this JSONB array; no separate items table needed.

4. **Schedule embedded on store** — stores.schedule_json stores the current week's predicted visitors, recommended staff, actual staff, and service level per hour. Updated daily by the forecasting pipeline.

5. **Basket rules embedded on store** — stores.basket_json carries the latest association rules with plain-language layout recommendations, surfaced directly in the store manager dashboard.

6. **Performance summary pre-computed** — stores.performance_json provides today/this-week/this-month summaries and YoY changes without aggregating traffic_counts and pos_transactions on every dashboard load.

7. **Context overlays on store** — weather, promotions, and events embed as an array on the store, enabling the contextual overlay on the traffic trend chart.

8. **Privacy by design** — no customer PII in any table. Traffic counts are anonymous aggregate numbers. Transaction staff_name is for internal performance tracking only.
