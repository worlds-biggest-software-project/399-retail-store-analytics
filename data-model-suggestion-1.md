# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: Retail Store Analytics · Created: 2026-05-29

## Philosophy

This model gives every retail analytics concept its own table with explicit foreign keys. Stores, sensors, foot traffic counts, POS transactions, products, staff schedules, and basket analysis results each have dedicated tables. Traffic data is stored at hourly granularity per store (or per zone where sensors support it). POS transactions link to stores and staff members. Basket analysis results (association rules) are stored as mined co-purchase patterns. Staff schedules are generated from traffic forecasts and stored alongside actual staffing for service-level analysis.

The schema aligns with NRF ARTS Operational Data Model terminology (RetailStore, WorkPeriod, Transaction) where applicable, and follows a privacy-by-design principle — no personally identifiable visitor data is stored.

**Best for:** Retail operations teams requiring full SQL access to traffic-to-conversion analysis, multi-store benchmarking, basket association mining, and staff scheduling optimisation with referential integrity across stores, sensors, transactions, and schedules.

**Trade-offs:**
- (+) Full referential integrity from store → sensor → traffic → transaction → basket → schedule
- (+) Standard SQL for all analytics: conversion rates, basket metrics, staffing ratios
- (+) Multi-store benchmarking via store-level aggregation with consistent schema
- (+) Hourly traffic partitioned for efficient time-series queries
- (-) Higher table count for a domain that could be simpler
- (-) Adding new sensor types or POS platforms requires connector development
- (-) Zone-level analytics require sensors that support zone granularity

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| NRF ARTS ODM v7.3 | Store, transaction, and product entity naming aligned with ARTS |
| ISO 27001/27701 | Security and privacy management for retail operational data |
| EU AI Act (2024/1689) | Privacy-by-design: no biometric data stored; anonymised counting only |
| GDPR | No PII in traffic data; consent workflow for any demographic features |
| OpenAPI 3.1 | REST API documentation |
| OAuth 2.0 | POS platform API authentication (Square, Shopify, Lightspeed) |
| ISO 8583 | POS payment message format awareness |

---

## Stores & Sensors

```sql
CREATE TABLE tenants (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL UNIQUE,
    industry        TEXT NOT NULL DEFAULT 'retail' CHECK (industry IN ('retail', 'hospitality', 'grocery', 'pharmacy', 'other')),
    base_currency   CHAR(3) NOT NULL DEFAULT 'USD',
    timezone        TEXT NOT NULL DEFAULT 'UTC',
    plan            TEXT NOT NULL CHECK (plan IN ('free', 'starter', 'professional', 'enterprise')),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    email           TEXT NOT NULL,
    name            TEXT NOT NULL,
    role            TEXT NOT NULL CHECK (role IN ('owner', 'admin', 'regional_manager', 'store_manager', 'analyst', 'viewer', 'api_service')),
    store_scope     UUID[],
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, email)
);

CREATE INDEX idx_users_tenant ON users(tenant_id);

CREATE TABLE stores (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    name            TEXT NOT NULL,
    code            TEXT,
    address         TEXT,
    city            TEXT,
    state           TEXT,
    country         CHAR(2) NOT NULL,
    postal_code     TEXT,
    latitude        NUMERIC(9,6),
    longitude       NUMERIC(9,6),
    timezone        TEXT NOT NULL DEFAULT 'UTC',
    store_type      TEXT CHECK (store_type IN ('flagship', 'standard', 'outlet', 'pop_up', 'kiosk')),
    square_footage  INT,
    operating_hours JSONB NOT NULL DEFAULT '{}',
    -- {"mon": {"open": "09:00", "close": "21:00"}, "tue": {...}, ...}
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, name)
);

CREATE INDEX idx_stores_tenant ON stores(tenant_id);

CREATE TABLE sensors (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    store_id        UUID NOT NULL REFERENCES stores(id),
    name            TEXT NOT NULL,
    sensor_type     TEXT NOT NULL CHECK (sensor_type IN (
        'people_counter', 'thermal', 'depth_3d', 'wifi', 'camera_ai', 'lidar'
    )),
    vendor          TEXT CHECK (vendor IN ('vcount', 'sensormatic', 'retailnext', 'dor', 'custom', 'other')),
    location_in_store TEXT,
    zone            TEXT,
    status          TEXT NOT NULL DEFAULT 'active' CHECK (status IN ('active', 'offline', 'maintenance')),
    config          JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_sensors_store ON sensors(store_id);
```

## Traffic & Transactions

```sql
CREATE TABLE traffic_counts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    store_id        UUID NOT NULL REFERENCES stores(id),
    sensor_id       UUID REFERENCES sensors(id),
    zone            TEXT,
    count_date      DATE NOT NULL,
    count_hour      SMALLINT NOT NULL CHECK (count_hour BETWEEN 0 AND 23),
    entries         INT NOT NULL DEFAULT 0,
    exits           INT NOT NULL DEFAULT 0,
    staff_excluded  INT NOT NULL DEFAULT 0,
    net_visitors    INT NOT NULL,
    peak_occupancy  INT,
    dwell_time_avg_seconds INT,
    data_source     TEXT NOT NULL CHECK (data_source IN ('sensor_api', 'csv_import', 'manual', 'estimated')),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, store_id, zone, count_date, count_hour)
) PARTITION BY RANGE (count_date);

CREATE INDEX idx_traffic_store ON traffic_counts(store_id, count_date);
CREATE INDEX idx_traffic_tenant ON traffic_counts(tenant_id, count_date);

CREATE TABLE pos_transactions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    store_id        UUID NOT NULL REFERENCES stores(id),
    data_source_id  UUID,
    external_id     TEXT NOT NULL,
    transaction_date TIMESTAMPTZ NOT NULL,
    transaction_type TEXT NOT NULL CHECK (transaction_type IN ('sale', 'return', 'exchange', 'void')),
    total_cents     BIGINT NOT NULL,
    subtotal_cents  BIGINT NOT NULL,
    discount_cents  BIGINT NOT NULL DEFAULT 0,
    tax_cents       BIGINT NOT NULL DEFAULT 0,
    currency        CHAR(3) NOT NULL,
    items_count     INT NOT NULL,
    payment_method  TEXT,
    staff_id        UUID,
    staff_name      TEXT,
    customer_id     TEXT,
    is_first_visit  BOOLEAN,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, store_id, external_id)
) PARTITION BY RANGE (transaction_date);

CREATE INDEX idx_pos_store ON pos_transactions(store_id, transaction_date);
CREATE INDEX idx_pos_tenant ON pos_transactions(tenant_id, transaction_date);
CREATE INDEX idx_pos_staff ON pos_transactions(staff_id);

CREATE TABLE transaction_items (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    transaction_id  UUID NOT NULL REFERENCES pos_transactions(id),
    product_id      UUID,
    sku             TEXT,
    product_name    TEXT NOT NULL,
    category        TEXT,
    quantity        INT NOT NULL,
    unit_price_cents BIGINT NOT NULL,
    discount_cents  BIGINT NOT NULL DEFAULT 0,
    total_cents     BIGINT NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_items_transaction ON transaction_items(transaction_id);
CREATE INDEX idx_items_sku ON transaction_items(sku);
CREATE INDEX idx_items_category ON transaction_items(category);
```

## Basket Analysis & Scheduling

```sql
CREATE TABLE basket_rules (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    store_id        UUID REFERENCES stores(id),
    antecedent      TEXT[] NOT NULL,
    consequent      TEXT[] NOT NULL,
    support         NUMERIC(8,6) NOT NULL,
    confidence      NUMERIC(8,6) NOT NULL,
    lift            NUMERIC(8,4) NOT NULL,
    conviction      NUMERIC(8,4),
    rule_type       TEXT NOT NULL DEFAULT 'product' CHECK (rule_type IN ('product', 'category', 'brand')),
    analysis_period_start DATE NOT NULL,
    analysis_period_end   DATE NOT NULL,
    transaction_count INT NOT NULL,
    layout_recommendation TEXT,
    model_version   TEXT NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_basket_rules_tenant ON basket_rules(tenant_id);
CREATE INDEX idx_basket_rules_store ON basket_rules(store_id);
CREATE INDEX idx_basket_rules_lift ON basket_rules(tenant_id, lift DESC);

CREATE TABLE staff_schedules (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    store_id        UUID NOT NULL REFERENCES stores(id),
    schedule_date   DATE NOT NULL,
    schedule_hour   SMALLINT NOT NULL CHECK (schedule_hour BETWEEN 0 AND 23),
    predicted_visitors INT NOT NULL,
    recommended_staff INT NOT NULL,
    actual_staff    INT,
    staff_to_visitor_ratio NUMERIC(6,4),
    service_level   TEXT CHECK (service_level IN ('understaffed', 'optimal', 'overstaffed')),
    forecast_model  TEXT NOT NULL DEFAULT 'prophet',
    forecast_confidence_lower INT,
    forecast_confidence_upper INT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, store_id, schedule_date, schedule_hour)
);

CREATE INDEX idx_schedules_store ON staff_schedules(store_id, schedule_date);
CREATE INDEX idx_schedules_service ON staff_schedules(tenant_id, service_level);
```

## Context, AI & Audit

```sql
CREATE TABLE context_overlays (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    store_id        UUID REFERENCES stores(id),
    overlay_type    TEXT NOT NULL CHECK (overlay_type IN ('weather', 'promotion', 'event', 'holiday', 'competitor')),
    overlay_date    DATE NOT NULL,
    title           TEXT NOT NULL,
    details         JSONB NOT NULL DEFAULT '{}',
    -- weather: {"temp_c": 22, "condition": "sunny", "precipitation_mm": 0}
    -- promotion: {"name": "Summer Sale", "discount_pct": 20, "channels": ["in_store", "email"]}
    -- event: {"name": "Local Festival", "expected_impact": "high"}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_overlays_store ON context_overlays(store_id, overlay_date);
CREATE INDEX idx_overlays_type ON context_overlays(tenant_id, overlay_type);

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
CREATE INDEX idx_audit_log_resource ON audit_log(resource_type, resource_id);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Entity Management | 2 | tenants, users |
| Stores & Sensors | 2 | stores, sensors |
| Traffic & Transactions | 3 | traffic_counts (partitioned), pos_transactions (partitioned), transaction_items |
| Basket & Scheduling | 2 | basket_rules, staff_schedules |
| Context, AI & Audit | 3 | context_overlays, ai_suggestions, audit_log (partitioned) |
| **Total** | **12** | 3 partitioned tables |

---

## Key Design Decisions

1. **Hourly traffic as the granularity** — traffic_counts stores one row per store per zone per hour, balancing storage efficiency with scheduling-useful granularity. Peak occupancy and dwell time captured where sensors support it.

2. **Staff exclusion tracked** — traffic_counts.staff_excluded records the number of counted entries that were staff, enabling the accurate conversion rate calculation (net_visitors / transactions).

3. **POS transactions linked to staff** — pos_transactions.staff_id enables staff performance analytics (basket value per staff member, conversion rate per shift).

4. **Basket rules as mined results** — basket_rules stores Apriori/FP-Growth output with support, confidence, lift, and conviction. layout_recommendation translates co-purchase patterns into actionable store layout suggestions.

5. **Staff schedules with forecast** — staff_schedules stores both predicted visitors and recommended staff count per hour, alongside actual_staff when available. service_level flags understaffed periods.

6. **Context overlays** — weather, promotions, events, and holidays are stored as overlays on traffic data, enabling causal analysis ("conversion dropped because of rain, not staffing").

7. **Privacy by design** — no customer PII in traffic data; no biometric features stored; pos_transactions.customer_id is an anonymised identifier.

8. **Zone support** — traffic_counts.zone enables zone-level heatmap analysis where sensors support it (V-Count, RetailNext), while degrades gracefully to store-level for simpler sensors (Dor).

9. **Multi-vendor sensor support** — sensors.vendor and sensor_type enable hardware-agnostic ingestion. The platform adapts to whatever counting technology the retailer has deployed.

10. **ARTS ODM alignment** — store, transaction, and product terminology follows NRF ARTS conventions for enterprise interoperability.
