# Retail Store Analytics — Phased Development Plan

> Project: 399-retail-store-analytics · Created: 2026-05-30
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

This plan synthesises `research.md`, `features.md`, `standards.md`, `README.md`, and the three data-model proposals. It adopts **data-model-suggestion-1 (Entity-Centric Normalized Relational)** as the canonical schema: it is the most directly implementable, aligns with NRF ARTS ODM terminology, supports every MVP analytics query with plain SQL, and degrades gracefully from zone-level to store-level granularity. JSONB columns from that model cover the configuration flexibility that data-model-suggestion-2 sought, and the `audit_log` + `ai_suggestions` tables capture the temporal-record value that data-model-suggestion-3 emphasised — without the operational complexity of full event sourcing.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Language | Python 3.12 | The entire analytics stack the research mandates — mlxtend (Apriori/FP-Growth), Prophet, Statsmodels, pandas — is Python-native. A single language for API, ingestion, and ML avoids a polyglot split between a TS API and a Python worker. |
| API framework | FastAPI | Generates OpenAPI 3.1 (a named standard requirement) automatically, has first-class Pydantic validation matching JSON Schema 2020-12, native async for webhook ingestion, and dependency-injection that cleanly enforces multi-tenant scoping (OWASP API #1 BOLA). |
| ASGI server | Uvicorn (dev) / Gunicorn+Uvicorn workers (prod) | Standard FastAPI deployment. |
| Database | PostgreSQL 16 | Data-model-1 relies on PostgreSQL-specific features: `UUID`, `JSONB`, `INET`, array columns (`TEXT[]`, `UUID[]`), declarative range partitioning, and partial indexes. PostgreSQL also covers the analytical workload at SMB scale without Snowflake's ops cost. |
| ORM / DB access | SQLAlchemy 2.0 (async) + Alembic | Async ORM matches FastAPI; Alembic manages migrations and the partition DDL that raw schema files cannot express incrementally. |
| Task queue | Celery + Redis | Async workloads (POS polling, nightly basket mining, forecast generation, webhook fan-out, alert dispatch) need durable scheduling. Celery Beat replaces the heavier Apache Airflow for SMB-scale cron-style orchestration; Airflow is noted as a backlog option for large estates. |
| Caching / broker | Redis 7 | Celery broker + result backend, plus dashboard query cache and rate-limit counters. |
| Time-series forecasting | Prophet (primary), Statsmodels (SARIMA fallback) | Named in research; Prophet handles seasonality + holiday regressors with minimal tuning, ideal for hourly traffic with weekly/daily cycles. |
| Basket analysis | mlxtend | Named in research; provides Apriori and FP-Growth + association_rules with support/confidence/lift/conviction matching the `basket_rules` columns. |
| Data transforms | pandas (in-worker); dbt-postgres for multi-store metric standardisation | dbt builds the benchmarking marts (consistent conversion/basket metrics across stores) the research calls for. |
| LLM provider | Anthropic Claude (via official SDK) with prompt caching | Powers natural-language query, plain-language layout recommendations, and anomaly narration. Provider abstracted behind an interface so it is swappable. |
| MCP server | `mcp` Python SDK | standards.md flags MCP as an unserved differentiator: expose store metrics/trends/scheduling to AI assistants without building a full NLP UI. |
| Frontend | Next.js 15 (React, TypeScript) + Tailwind + shadcn/ui | Store-manager dashboards for non-technical users; SSR for fast first paint. Recharts for trend charts; D3.js + Leaflet for zone heatmaps and multi-store geographic comparison (per README). |
| Auth | OAuth 2.0 (RFC 6749) for POS connectors; JWT sessions + OIDC for users | POS platforms (Square, Shopify, Lightspeed) all require OAuth 2.0; OIDC enables enterprise SSO (Azure AD/Okta) per standards.md. |
| Alerting | Slack Incoming Webhooks + Twilio SDK | Named in research for anomaly/queue alerts. |
| Containerisation | Docker + docker-compose | Self-hosted is an explicit deployment target; compose wires Postgres, Redis, API, worker, beat, frontend. |
| Testing | pytest + pytest-asyncio + httpx + testcontainers | Unit + integration; testcontainers spins real Postgres/Redis for integration tests. Vitest + Playwright for frontend. |
| Code quality | Ruff (lint+format), mypy (strict), pre-commit | Standard modern Python toolchain. |
| Package manager | uv | Fast, lockfile-based dependency resolution. |
| CI | GitHub Actions | Lint → type-check → test → docker build gates. |

### Project Structure

```
retail-store-analytics/
├── pyproject.toml
├── uv.lock
├── Dockerfile
├── docker-compose.yml
├── .pre-commit-config.yaml
├── alembic.ini
├── README.md
├── .env.example
├── migrations/                      # Alembic versions (incl. partition DDL)
│   └── versions/
├── dbt/                             # multi-store metric standardisation
│   ├── dbt_project.yml
│   └── models/
│       ├── staging/
│       └── marts/                   # store_daily_metrics, store_benchmark
├── src/
│   └── rsa/
│       ├── __init__.py
│       ├── main.py                  # FastAPI app factory
│       ├── config.py                # Pydantic Settings
│       ├── db/
│       │   ├── session.py
│       │   ├── base.py
│       │   └── models/              # SQLAlchemy ORM models (one per table)
│       ├── schemas/                 # Pydantic request/response models
│       ├── api/
│       │   ├── deps.py              # auth, tenant scoping, pagination
│       │   ├── routes/              # stores, traffic, pos, basket, schedules,
│       │   │                        # benchmark, alerts, nlq, webhooks, auth
│       │   └── errors.py
│       ├── core/
│       │   ├── security.py          # JWT, password, API keys
│       │   ├── tenancy.py           # tenant-scoped query helpers (BOLA guard)
│       │   ├── rbac.py              # role + store_scope enforcement
│       │   └── audit.py
│       ├── ingestion/
│       │   ├── sensors/             # webhook + CSV importers, vendor adapters
│       │   ├── pos/                 # square.py, shopify.py, lightspeed.py
│       │   └── normalise.py         # ARTS-aligned canonical mapping
│       ├── analytics/
│       │   ├── conversion.py
│       │   ├── basket.py            # mlxtend wrapper
│       │   ├── forecast.py          # Prophet wrapper
│       │   ├── scheduling.py        # roster recommender
│       │   ├── benchmark.py
│       │   └── anomaly.py
│       ├── ai/
│       │   ├── provider.py          # LLM interface + Anthropic impl
│       │   ├── nlq.py               # natural-language → query plan
│       │   ├── narratives.py        # layout recs, anomaly explanations
│       │   └── prompts/             # templated system/user prompts
│       ├── integrations/
│       │   ├── slack.py
│       │   ├── twilio.py
│       │   └── weather.py
│       ├── tasks/                   # Celery tasks + beat schedule
│       │   ├── celery_app.py
│       │   ├── poll_pos.py
│       │   ├── nightly_basket.py
│       │   ├── nightly_forecast.py
│       │   └── alerts.py
│       └── mcp/
│           └── server.py            # MCP server exposing analytics tools
├── tests/
│   ├── conftest.py
│   ├── fixtures/                    # sample CSVs, POS payloads, webhook bodies
│   ├── unit/
│   ├── integration/
│   └── e2e/
└── frontend/
    ├── package.json
    ├── app/                         # Next.js App Router
    ├── components/
    └── lib/
```

The structure is grouped by concern (ingestion, analytics, ai, integrations) so every phase adds files within existing packages rather than restructuring.

---

## Phase 1: Foundation — Project, Config, Multi-Tenant Data Layer

### Purpose
Stand up the skeleton that every later phase depends on: dependency management, configuration, the PostgreSQL schema from data-model-1 via Alembic, the SQLAlchemy models, and the multi-tenant scoping primitives that prevent cross-tenant data leakage (OWASP API #1). After this phase the app boots, connects to a real database, and exposes a health endpoint, but does no analytics yet.

### Tasks

#### 1.1 — Project scaffold & tooling

**What**: Initialise the repo with uv, Ruff, mypy, pre-commit, Docker, and docker-compose.

**Design**:
- `pyproject.toml` declares dependencies: `fastapi`, `uvicorn[standard]`, `sqlalchemy[asyncio]`, `asyncpg`, `alembic`, `pydantic-settings`, `celery[redis]`, `redis`, `python-jose[cryptography]`, `passlib[bcrypt]`, `httpx`, plus dev group `pytest pytest-asyncio testcontainers ruff mypy`.
- `config.py` — Pydantic `Settings`:
```python
class Settings(BaseSettings):
    database_url: str
    redis_url: str = "redis://localhost:6379/0"
    jwt_secret: str
    jwt_algorithm: str = "HS256"
    access_token_ttl_minutes: int = 60
    environment: Literal["dev", "test", "prod"] = "dev"
    anthropic_api_key: str | None = None
    model_config = SettingsConfigDict(env_file=".env", env_prefix="RSA_")
```
- `docker-compose.yml` services: `db` (postgres:16), `redis` (redis:7), `api`, `worker`, `beat`, `frontend`. `api` depends_on `db` + `redis`.
- `main.py` exposes `GET /health` returning `{"status": "ok", "db": <bool>}` (db check via `SELECT 1`).

**Testing**:
- `Unit: Settings loads from env vars with RSA_ prefix → correct typed values`
- `Unit: missing required jwt_secret → ValidationError naming the field`
- `Integration: GET /health with DB up → 200 {"status":"ok","db":true}`
- `Integration: GET /health with DB down → 200 {"status":"ok","db":false}` (degraded, not crash)

#### 1.2 — Schema migration (data-model-1)

**What**: Alembic migration creating all 12 tables from data-model-suggestion-1, including partitioning and indexes.

**Design**:
- Tables exactly as specified in data-model-1: `tenants`, `users`, `stores`, `sensors`, `traffic_counts` (RANGE partition on `count_date`), `pos_transactions` (RANGE partition on `transaction_date`), `transaction_items`, `basket_rules`, `staff_schedules`, `context_overlays`, `ai_suggestions`, `audit_log` (RANGE partition on `created_at`).
- Partitioned tables: create parent + a helper SQL function `create_monthly_partition(table, month)` invoked by a Celery beat task and by tests. Initial migration creates partitions for the current and next month.
- All `CHECK` constraints, `UNIQUE` constraints, and indexes from the data model are reproduced verbatim.
- Enable `pgcrypto` extension for `gen_random_uuid()`.

**Testing**:
- `Integration (testcontainers): alembic upgrade head → all 12 tables present (query information_schema)`
- `Integration: insert into traffic_counts with count_hour=24 → CHECK violation`
- `Integration: duplicate (tenant_id, store_id, zone, count_date, count_hour) → unique violation`
- `Integration: alembic downgrade base → upgrade head round-trips cleanly`

#### 1.3 — ORM models & tenant scoping

**What**: SQLAlchemy 2.0 models and a `tenant_scoped()` query helper.

**Design**:
- One mapped class per table under `db/models/`, using `Mapped[...]` typing.
- `core/tenancy.py`:
```python
def tenant_scoped(stmt: Select, model, tenant_id: UUID) -> Select:
    return stmt.where(model.tenant_id == tenant_id)
```
- Every repository function takes `tenant_id` as a required first argument; a mypy plugin/test asserts no route handler queries a tenant-scoped table without it.

**Testing**:
- `Unit: tenant_scoped adds WHERE tenant_id = :id to the statement`
- `Integration: query stores for tenant A returns only A's stores when B's rows exist`

### Definition of Done
App boots in Docker; `alembic upgrade head` succeeds; health endpoint green; tenant-scoping helper covered by tests; Ruff + mypy clean.

---

## Phase 2: Identity, RBAC & API Surface Conventions

### Purpose
Add authentication, role-based access control with per-store scoping, audit logging, pagination, and error conventions. This is the security spine: every subsequent endpoint inherits these guards. RBAC enforces the `owner/admin/regional_manager/store_manager/analyst/viewer/api_service` roles and the `store_scope` array from the data model.

### Tasks

#### 2.1 — User auth (JWT + API keys)

**What**: Login, token issuance, password hashing, and service API keys.

**Design**:
- `POST /auth/login` body `{email, password}` → `{access_token, token_type:"bearer", expires_in}`. Passwords hashed with bcrypt (passlib).
- JWT claims: `sub` (user id), `tid` (tenant id), `role`, `scope` (store ids or `["*"]`), `exp`.
- API keys (for `api_service` role / sensor webhooks): stored hashed; passed as `X-API-Key`. Table reuse: API keys modelled as `users` rows with `role='api_service'` plus a `secrets` side table (added in this migration).
- `deps.get_current_principal()` decodes JWT or API key → `Principal(user_id, tenant_id, role, store_scope)`.

**Testing**:
- `Unit: valid password verifies; wrong password fails`
- `Integration: POST /auth/login valid → 200 with JWT whose tid matches user`
- `Integration: POST /auth/login bad password → 401, no token`
- `Integration: expired JWT → 401`

#### 2.2 — RBAC & store-scope enforcement

**What**: Dependency that checks role + store access on every protected route.

**Design**:
- `require_role(*roles)` and `require_store_access(store_id)` FastAPI dependencies.
- `core/rbac.py` matrix maps role → allowed actions. `store_manager` limited to stores in `store_scope`; `regional_manager`/`admin`/`owner` see all tenant stores; `viewer` read-only; `analyst` read + export.
- BOLA guard: `require_store_access` rejects (404, not 403, to avoid existence leak) when the requested `store_id` is not in scope or belongs to another tenant.

**Testing**:
- `Integration: store_manager scoped to store X requests store Y data → 404`
- `Integration: viewer attempts POST mutation → 403`
- `Integration: regional_manager reads any store in tenant → 200`
- `Integration: principal from tenant A requests tenant B store → 404 (BOLA)`

#### 2.3 — Audit log, pagination, error format

**What**: Cross-cutting middleware/utilities.

**Design**:
- `core/audit.py`: `record(tenant_id, actor_type, actor_id, action, resource_type, resource_id, changes, ip, ua)` inserts into `audit_log`. Called on all mutations.
- Pagination: cursor-based using RFC 8288 `Link` headers (`rel="next"`); query params `limit` (default 50, max 200) and `cursor`.
- Error envelope: `{"error": {"code": str, "message": str, "field": str|null}}` with a global exception handler mapping `ValidationError`→422, domain errors→4xx.

**Testing**:
- `Integration: any POST creating a store writes an audit_log row with action="store.create"`
- `Integration: list endpoint with 60 rows, limit=50 → 50 items + Link rel=next`
- `Unit: validation error serialises to error envelope with field name`

### Definition of Done
All routes require a principal; RBAC + store-scope tests pass; audit rows written on mutation; OpenAPI spec shows security schemes; mypy/Ruff clean.

---

## Phase 3: Sensor & Traffic Ingestion (Core Value, Part 1)

### Purpose
Ingest foot-traffic data — the heart of the product — via both a generic webhook receiver and CSV import, normalised into `traffic_counts` at hourly granularity with staff exclusion. Because no sensor vendor publishes open docs (per standards.md), the design centres on a **hardware-agnostic canonical format** plus thin vendor adapters.

### Tasks

#### 3.1 — Canonical traffic ingestion model & normaliser

**What**: A vendor-neutral payload schema and mapping into `traffic_counts`.

**Design**:
- Canonical Pydantic model:
```python
class TrafficReading(BaseModel):
    store_code: str
    zone: str | None = None
    timestamp: datetime          # event time, tz-aware
    entries: int
    exits: int = 0
    staff_excluded: int = 0
    peak_occupancy: int | None = None
    dwell_time_avg_seconds: int | None = None
    sensor_external_id: str | None = None
```
- `normalise.py` aggregates readings into hourly buckets (store timezone), computes `net_visitors = entries - staff_excluded`, and UPSERTs on the unique key `(tenant_id, store_id, zone, count_date, count_hour)` summing entries/exits.
- Vendor adapters (`ingestion/sensors/{vcount,sensormatic,dor,generic}.py`) map raw vendor JSON → `TrafficReading`. `data_source` set to `sensor_api`.

**Testing**:
- `Unit: two readings in same hour for same store → single hourly row with summed entries`
- `Unit: net_visitors = entries - staff_excluded`
- `Unit: reading in store tz crossing midnight buckets into correct local date/hour`
- `Unit: vcount adapter maps sample payload → TrafficReading`

#### 3.2 — Webhook receiver

**What**: Authenticated endpoint accepting sensor pushes, enqueuing normalisation.

**Design**:
- `POST /webhooks/traffic/{vendor}` authenticated via `X-API-Key`; verifies HMAC signature header where the vendor supports it (`X-Signature`, HMAC-SHA256 over body with per-sensor secret).
- Body validated to the vendor's raw shape, mapped via adapter, then a Celery task `ingest_traffic.delay(tenant_id, readings)` persists. Returns `202 Accepted` immediately.
- Idempotency: dedupe on `(sensor_external_id, timestamp)` within the task.

**Testing**:
- `Integration (mocked queue): valid signed webhook → 202, task enqueued`
- `Integration: invalid signature → 401, no task enqueued`
- `Integration: unknown vendor path → 404`
- `Integration (real worker, testcontainers): webhook → row appears in traffic_counts`

#### 3.3 — CSV import

**What**: Bulk historical traffic import for hardware-agnostic deployments.

**Design**:
- `POST /stores/{store_id}/traffic/import` multipart CSV with header `timestamp,zone,entries,exits,staff_excluded`. Streamed parse, per-row validation, errors collected.
- Response: `{"imported": int, "skipped": int, "errors": [{"row": int, "message": str}]}`. `data_source='csv_import'`.

**Testing**:
- `Unit: well-formed CSV → N TrafficReadings`
- `Integration: CSV with one malformed row → that row in errors[], others imported`
- `E2E: upload fixture CSV → GET hourly traffic returns expected aggregates`

### Definition of Done
Traffic lands in `traffic_counts` from webhook + CSV; hourly aggregation + staff exclusion verified; idempotent re-ingestion; fixtures committed under `tests/fixtures/`.

---

## Phase 4: POS Ingestion & Conversion Analytics (Core Value, Part 2)

### Purpose
Ingest POS transactions and items via Square, Shopify, and Lightspeed (all OAuth 2.0), normalise to ARTS-aligned canonical transactions, and compute the conversion-rate metric — visitors-to-transactions with staff excluded — which is the platform's table-stakes deliverable.

### Tasks

#### 4.1 — POS OAuth connection management

**What**: Connect a tenant's POS account and store tokens.

**Design**:
- `data_sources` table (added migration): `id, tenant_id, store_id, platform (square|shopify|lightspeed), access_token (encrypted), refresh_token (encrypted), expires_at, status, external_account_id`.
- OAuth 2.0 authorization-code flow: `GET /connect/{platform}/start` → redirect; `GET /connect/{platform}/callback` → exchange code, encrypt tokens (Fernet via app key), persist. Token refresh handled in a helper before each API call.

**Testing**:
- `Integration (mocked OAuth server): callback with valid code → tokens stored encrypted`
- `Unit: expired token triggers refresh before request`
- `Integration: callback with state mismatch → 400 (CSRF guard)`

#### 4.2 — POS connectors & canonical transaction normaliser

**What**: Pull orders/transactions from each platform and map to `pos_transactions` + `transaction_items`.

**Design**:
- Connector interface:
```python
class POSConnector(Protocol):
    def fetch_transactions(self, since: datetime) -> Iterable[CanonicalTransaction]: ...
```
- `CanonicalTransaction` carries `external_id, store_code, transaction_date, type, total_cents, subtotal_cents, discount_cents, tax_cents, currency, items[], payment_method, staff_id, staff_name, customer_id`. ARTS terminology (Transaction, line items).
- Square: REST Orders API; Shopify: GraphQL Admin orders; Lightspeed: X-Series sales endpoint. All amounts normalised to integer cents.
- UPSERT on `(tenant_id, store_id, external_id)`; items inserted child-side. Edge cases per research: voids/returns set `transaction_type`; multi-tender → `payment_method='mixed'`; staff discounts retained in `discount_cents`.

**Testing**:
- `Unit: Square sample order → CanonicalTransaction with correct cents`
- `Unit: return order → transaction_type='return', negative not assumed (type-flagged)`
- `Unit: multi-tender → payment_method='mixed'`
- `Integration: re-fetch same order → no duplicate (UPSERT)`

#### 4.3 — Polling task & conversion analytics

**What**: Scheduled POS polling and the conversion-rate computation/endpoints.

**Design**:
- Celery beat task `poll_pos` runs every 15 min per active `data_source`, calling `fetch_transactions(since=last_sync)`.
- `analytics/conversion.py`:
```python
def conversion_rate(net_visitors: int, transactions: int) -> float:
    return 0.0 if net_visitors == 0 else transactions / net_visitors
```
- `GET /stores/{id}/metrics?granularity=hour|day|week&from=&to=` returns per-bucket `{net_visitors, transactions, basket_value_avg, items_per_txn, conversion_rate}` joining `traffic_counts` and `pos_transactions`.

**Testing**:
- `Unit: conversion_rate(0, x) → 0.0 (no divide-by-zero)`
- `Unit: 100 visitors, 25 sale txns → 0.25`
- `Integration: metrics endpoint joins traffic + pos correctly for a day`
- `Integration: returns/voids excluded from conversion numerator`

### Definition of Done
POS data flows in for all three platforms (real adapters, mocked APIs in CI); conversion endpoint returns correct figures; tokens encrypted at rest; polling task scheduled.

---

## Phase 5: Forecasting & AI Staff Scheduling (Primary Differentiator)

### Purpose
Deliver the closed-loop differentiator the README names: forecast hourly traffic with Prophet and translate forecasts into recommended rosters in `staff_schedules`, flagging understaffed/overstaffed periods. This is what no surveyed SMB tool does end-to-end.

### Tasks

#### 5.1 — Traffic forecasting

**What**: Per-store hourly traffic forecast with seasonality and context regressors.

**Design**:
- `analytics/forecast.py`:
```python
def forecast_traffic(history: pd.DataFrame, horizon_hours: int,
                     regressors: dict[str, pd.Series] | None = None) -> Forecast:
    # history: columns ds (hourly), y (net_visitors)
    # Prophet with daily+weekly seasonality; add holiday + weather regressors
    ...
@dataclass
class ForecastPoint:
    ds: datetime; yhat: int; yhat_lower: int; yhat_upper: int
```
- Holidays + promotions sourced from `context_overlays`; weather as a numeric regressor. SARIMA (statsmodels) fallback when history < 14 days.
- Nightly Celery task `nightly_forecast` produces next-7-day hourly forecasts per active store.

**Testing**:
- `Unit: synthetic weekly-seasonal series → forecast captures weekend peaks (yhat higher Sat/Sun)`
- `Unit: <14 days history → SARIMA fallback used`
- `Unit: forecast points clamp negatives to 0`
- `Integration: nightly task writes predicted_visitors into staff_schedules`

#### 5.2 — Roster recommendation

**What**: Convert predicted hourly visitors into recommended staff counts and service-level flags.

**Design**:
- Config per store (JSONB in `stores` or new `scheduling_config`): `visitors_per_staff` target (default 25), `min_staff` (default 1), `max_staff`, store operating hours.
- `analytics/scheduling.py`:
```python
def recommend_staff(predicted: int, cfg: SchedulingConfig) -> int:
    return clamp(ceil(predicted / cfg.visitors_per_staff), cfg.min_staff, cfg.max_staff)
def service_level(actual: int, recommended: int) -> Literal["understaffed","optimal","overstaffed"]:
    ...
```
- Writes `recommended_staff`, `staff_to_visitor_ratio`, `forecast_confidence_{lower,upper}` to `staff_schedules`. When `actual_staff` is later supplied, recompute `service_level`.
- `GET /stores/{id}/schedule?from=&to=` and `PUT /stores/{id}/schedule/{date}/{hour}` (set `actual_staff`).

**Testing**:
- `Unit: 80 predicted, target 25 → recommended 4 (ceil), clamped to max`
- `Unit: predicted 0 outside hours → min_staff respected`
- `Unit: actual 2 vs recommended 4 → "understaffed"`
- `Integration: schedule endpoint returns hourly recommendations for a date range`

### Definition of Done
Nightly forecast + roster recommendations populate `staff_schedules`; service-level flags correct; schedule endpoints live; forecasting handles cold-start stores.

---

## Phase 6: Market Basket Analysis (v1.1 Differentiator)

### Purpose
Mine product co-purchase association rules with mlxtend over `transaction_items`, store them in `basket_rules` with support/confidence/lift/conviction, and surface plain-language layout recommendations — closing the "basket linked to traffic" gap research identified as unserved.

### Tasks

#### 6.1 — Association-rule mining

**What**: Apriori/FP-Growth mining producing `basket_rules` rows.

**Design**:
- `analytics/basket.py`:
```python
def mine_rules(items: pd.DataFrame, *, min_support=0.01, min_confidence=0.3,
               algo: Literal["apriori","fpgrowth"]="fpgrowth",
               level: Literal["product","category","brand"]="product") -> list[BasketRule]:
    # items: rows = transaction_id, columns = product (one-hot via TransactionEncoder)
    # mlxtend fpgrowth → association_rules(metric="lift", min_threshold=1.0)
```
- Each rule maps to a `basket_rules` row: `antecedent[]`, `consequent[]`, support, confidence, lift, conviction, `transaction_count`, `analysis_period_{start,end}`, `model_version`.
- Nightly Celery task `nightly_basket` runs per store over a configurable trailing window (default 90 days); rules replaced for that period.

**Testing**:
- `Unit: transactions where bread→butter co-occur strongly → rule with lift>1`
- `Unit: min_support filters out rare itemsets`
- `Unit: category-level mining aggregates SKUs to category`
- `Integration: nightly_basket writes rules ordered by lift`

#### 6.2 — Plain-language layout recommendations

**What**: Convert top rules into actionable layout suggestions via the LLM.

**Design**:
- `ai/narratives.py` `layout_recommendation(rule)` builds a prompt (template in `ai/prompts/layout.txt`):
```
System: You are a retail merchandising advisor. Given a co-purchase association
rule, produce ONE concrete, plain-language layout or end-cap recommendation.
Be specific, avoid jargon, do not claim causation.
User: Customers who buy {antecedent} also buy {consequent}. lift={lift},
confidence={confidence}, over {transaction_count} transactions.
```
- Result stored in `basket_rules.layout_recommendation`. Prompt caching on the system block. Causation caveat enforced by the prompt (addresses the "correlation not causation" challenge).
- `GET /stores/{id}/basket-rules?min_lift=&limit=` returns rules + recommendations.

**Testing**:
- `Integration (mocked LLM): rule → recommendation string persisted`
- `Unit: prompt template fills antecedent/consequent/metrics`
- `Integration: basket-rules endpoint filters by min_lift, sorts desc`

### Definition of Done
Rules mined nightly with correct metrics; layout recommendations generated and stored; endpoint returns ranked rules; mlxtend integration covered by fixture tests.

---

## Phase 7: Context Overlays, Anomaly Detection & Alerting (v1.1)

### Purpose
Add weather/promotion/event/holiday overlays, AI anomaly detection (traffic spikes, conversion drops, queue/service-level breaches), and Slack/Twilio alerting — making the platform proactive rather than passive, and improving forecast accuracy via regressors.

### Tasks

#### 7.1 — Context overlays

**What**: Ingest and store weather/promotion/event/holiday context.

**Design**:
- `integrations/weather.py` fetches daily weather per store lat/long (pluggable provider) → `context_overlays` rows (`overlay_type='weather'`, details JSONB).
- `POST /stores/{id}/overlays` for manual promotions/events. These feed forecast regressors (Phase 5) and anomaly explanations.

**Testing**:
- `Integration (mocked weather API): daily fetch writes weather overlay`
- `Integration: POST promotion overlay → row with details JSONB`

#### 7.2 — Anomaly detection

**What**: Detect statistically unusual traffic and conversion behaviour, recording `ai_suggestions`.

**Design**:
- `analytics/anomaly.py`: compares observed hourly traffic/conversion against forecast confidence band and historical z-score. Triggers:
  - `traffic_anomaly`: observed outside `[yhat_lower, yhat_upper]` by > configured margin.
  - `conversion_drop`: day conversion > 2σ below trailing-28-day mean.
  - `queue_alert` / `staffing_alert`: `service_level='understaffed'` for ≥ N consecutive hours.
- Each writes an `ai_suggestions` row (`severity`, `evidence` JSONB, `recommended_action`, `confidence`). The LLM generates a short `description` explaining likely cause using overlapping context overlays (weather/promo).

**Testing**:
- `Unit: observed 3× upper band → traffic_anomaly suggestion (severity warning)`
- `Unit: conversion 2.5σ below mean → conversion_drop suggestion`
- `Unit: no anomaly when within band → no suggestion`
- `Integration (mocked LLM): suggestion description references overlapping weather overlay`

#### 7.3 — Alert dispatch

**What**: Push warning/critical suggestions to Slack and Twilio.

**Design**:
- `notification_channels` table (migration): `tenant_id, store_id, kind (slack|sms), config JSONB (webhook_url or phone), min_severity`.
- Celery task `dispatch_alerts` runs after anomaly detection; sends matching suggestions; records delivery in `audit_log`. Dedup: one alert per suggestion id.

**Testing**:
- `Integration (mocked Slack): critical suggestion → POST to webhook with formatted message`
- `Integration (mocked Twilio): SMS sent only when severity ≥ channel min_severity`
- `Unit: same suggestion not dispatched twice`

### Definition of Done
Overlays stored and feeding forecasts; anomalies generate suggestions with explanations; alerts dispatched per channel config with severity filtering; all external calls mocked in CI.

---

## Phase 8: Multi-Store Benchmarking & dbt Marts

### Purpose
Standardise operational metrics across stores so regional managers can compare on a like-for-like basis — a table-stakes feature. dbt builds the marts; the API exposes ranked benchmarks.

### Tasks

#### 8.1 — dbt metric marts

**What**: dbt models producing consistent per-store daily metrics.

**Design**:
- `dbt/models/marts/store_daily_metrics.sql`: per store/day `net_visitors, transactions, conversion_rate, avg_basket_value_cents, items_per_txn, recommended_vs_actual_staff_gap`.
- `store_benchmark.sql`: ranks stores within a tenant per metric over a period (percentile + rank). Materialised as tables, refreshed by a Celery task invoking `dbt run`.

**Testing**:
- `Integration (testcontainers + dbt): seed sample data → store_daily_metrics matches hand-computed values`
- `Integration: benchmark ranks higher-conversion store above lower`

#### 8.2 — Benchmark API

**What**: Endpoints for cross-store comparison.

**Design**:
- `GET /benchmark?metric=conversion_rate&from=&to=` → tenant-scoped ranked list `[{store_id, name, value, rank, percentile}]`. RBAC: `regional_manager`+ only.
- `GET /benchmark/{store_id}` → that store's metrics vs tenant median.

**Testing**:
- `Integration: store_manager (single-store scope) → 403 on tenant-wide benchmark`
- `Integration: regional_manager → ranked list across all stores`
- `Unit: percentile computed correctly for small N`

### Definition of Done
dbt marts build in CI; benchmark endpoints return correct rankings; RBAC restricts cross-store views; marts refreshed on schedule.

---

## Phase 9: Natural-Language Query & MCP Server (AI-Native Layer)

### Purpose
Let store managers ask questions in plain English ("Why was last Saturday's conversion 3% below average?") without SQL, and expose the same analytics to external AI assistants via MCP — a differentiator no surveyed vendor offers.

### Tasks

#### 9.1 — Natural-language query

**What**: Translate NL questions into safe, parameterised analytics queries.

**Design**:
- `ai/nlq.py`: the LLM is given a constrained tool schema (not free SQL) — a set of named query intents (`get_metrics`, `compare_periods`, `explain_anomaly`, `top_basket_rules`, `staffing_for_date`) with typed params. LLM returns a structured intent; the backend executes the corresponding repository function (tenant + store-scope enforced). The LLM then narrates the result.
- `POST /nlq` body `{question, store_id?}` → `{answer, data, intent}`. Never executes raw LLM-authored SQL (injection/BOLA safety).

**Testing**:
- `Unit: "conversion last Saturday vs average" → compare_periods intent with correct dates`
- `Integration (mocked LLM): intent dispatched to repo fn, narrated answer returned`
- `Integration: question referencing out-of-scope store → 404 before LLM narration`

#### 9.2 — MCP server

**What**: MCP server exposing analytics tools to AI assistants.

**Design**:
- `mcp/server.py` registers tools mirroring the NLQ intents: `get_store_metrics`, `get_traffic_trend`, `get_schedule_recommendation`, `get_basket_rules`, `list_open_alerts`. Each authenticates via API key → resolves `Principal` → tenant/store scoping applies identically to the REST API.
- Tools return structured JSON; descriptions written for LLM consumption (per modelcontextprotocol.io).

**Testing**:
- `Integration: MCP get_store_metrics with valid key → metrics for in-scope store`
- `Integration: MCP tool with key scoped to store X requesting store Y → error, no data`
- `Unit: tool registry lists all five tools with schemas`

### Definition of Done
NLQ answers common questions safely with no raw-SQL path; MCP server runs and enforces the same tenancy/scoping; both covered by mocked-LLM tests.

---

## Phase 10: Dashboards & Heatmaps (Store-Manager UI)

### Purpose
Deliver the non-technical store-manager interface: traffic/conversion dashboards, schedule views, basket recommendations, benchmarking, zone heatmaps, and an NLQ chat box. Built last because it consumes every prior endpoint.

### Tasks

#### 10.1 — Core dashboards

**What**: Next.js app with auth and the primary operational views.

**Design**:
- App Router pages: `/login`, `/stores/[id]` (traffic + conversion trend via Recharts, PoP toggles day/week/month/year), `/stores/[id]/schedule`, `/stores/[id]/basket`, `/benchmark`.
- Server components fetch via the REST API using the session JWT; role gates hide cross-store views from `store_manager`.

**Testing**:
- `E2E (Playwright): login → store dashboard shows traffic chart with seeded data`
- `E2E: store_manager cannot see /benchmark (redirect/hidden)`
- `Component (Vitest): conversion trend renders PoP comparison`

#### 10.2 — Zone heatmap & geographic comparison

**What**: D3/Leaflet visualisations.

**Design**:
- Zone heatmap: D3 rendering `traffic_counts` aggregated by `zone` over a store floorplan SVG; degrades to a single-zone bar when sensors lack zone support (Dor case).
- Leaflet map for multi-store geographic benchmarking using `stores.latitude/longitude`, colour-coded by selected metric.

**Testing**:
- `Component: heatmap renders N zones with intensity proportional to net_visitors`
- `Component: store with no zone data → fallback view, no crash`
- `E2E: benchmark map plots stores at correct coordinates`

#### 10.3 — NLQ chat & alerts inbox

**What**: Chat box wired to `/nlq` and an alerts panel over `ai_suggestions`.

**Design**:
- Chat posts to `/nlq`, renders answer + any returned data table/chart.
- Alerts inbox lists `ai_suggestions` (pending first), with accept/dismiss actions (`PATCH /suggestions/{id}`) writing `status` + `resolved_at`.

**Testing**:
- `E2E (mocked LLM): ask question → answer rendered`
- `E2E: dismiss alert → status updates, row leaves pending list`

### Definition of Done
Manager can log in and operate the full workflow in-browser; heatmaps degrade gracefully; NLQ + alerts functional; Playwright e2e suite green.

---

## Phase 11: Privacy, Compliance & Hardening

### Purpose
Make GDPR / EU AI Act / privacy-by-design architectural guarantees real and verifiable, and harden the API against the OWASP API Security Top 10 — turning compliance into a selling point rather than buyer responsibility, ahead of the August 2026 EU AI Act enforcement noted in standards.md.

### Tasks

#### 11.1 — Privacy guarantees & data retention

**What**: Enforce no-PII storage and configurable retention.

**Design**:
- A schema-level test/assertion confirms no table stores biometric or raw-image data; `pos_transactions.customer_id` is documented as an opaque anonymised token only.
- Retention: per-tenant config; Celery task drops aged `traffic_counts`/`audit_log` partitions and purges raw POS beyond retention. Data export (`GET /tenants/{id}/export`) and deletion endpoints support DSR-style requests.
- A `PRIVACY.md` records lawful basis, retention defaults, and the EU AI Act transparency posture (non-biometric counting → Article 50 transparency only).

**Testing**:
- `Integration: retention task drops partitions older than configured window`
- `Integration: export returns tenant data in JSON; no PII fields present`
- `Test: schema scan asserts absence of biometric/image columns`

#### 11.2 — Security hardening (OWASP API Top 10)

**What**: Rate limiting, BOLA regression suite, secret handling.

**Design**:
- Redis-backed rate limiter per principal + per IP on auth and webhook routes.
- A dedicated BOLA regression test set hitting every object-scoped route cross-tenant and cross-store.
- Secrets (POS tokens, API keys) encrypted at rest; security headers; input size limits on uploads/webhooks.

**Testing**:
- `Integration: exceeding rate limit → 429`
- `Integration: cross-tenant access on every resource route → 404 (BOLA suite)`
- `Integration: oversized CSV/webhook body → 413`

### Definition of Done
Privacy guarantees test-enforced; retention + export/delete working; OWASP API Top 10 checklist addressed; BOLA regression suite green; PRIVACY.md published.

---

## Phase 12: Packaging, Deployment & Operations

### Purpose
Make the platform deployable self-hosted and to the cloud with confidence: production Docker images, migrations, partition maintenance, observability, backups, and seed data.

### Tasks

#### 12.1 — Production deployment

**What**: Hardened images and compose/profiles for prod.

**Design**:
- Multi-stage `Dockerfile` (builder → slim runtime), non-root user. `docker-compose.prod.yml` with Gunicorn+Uvicorn workers, separate `worker`/`beat`, healthchecks, and a migration init container running `alembic upgrade head`.
- `.env.example` documents every setting; startup fails fast on missing required secrets.

**Testing**:
- `CI: docker build succeeds; image runs and passes /health`
- `Integration: fresh compose up → migrations applied, health green`

#### 12.2 — Operations: partitions, observability, backups, seed

**What**: Keep the system healthy in production.

**Design**:
- Celery beat task `ensure_partitions` creates next-month partitions for `traffic_counts`/`pos_transactions`/`audit_log` ahead of need.
- Structured JSON logging with request/trace ids; `/metrics` Prometheus endpoint (request counts, task durations, ingestion lag).
- `pg_dump` backup task + restore runbook. `seed.py` loads a demo tenant with synthetic stores/traffic/POS so the dashboard is populated on first run.

**Testing**:
- `Integration: ensure_partitions creates next month's partition`
- `Integration: seed script populates a working demo tenant`
- `Unit: log records include trace id; /metrics exposes counters`

### Definition of Done
Prod images build and run; migrations + partition maintenance automated; logging/metrics/backups in place; seed produces a populated demo; CI green end-to-end.

---

## Phase Summary & Dependencies

```
Phase 1: Foundation (schema, tenancy)        ─── required by everything
    │
Phase 2: Identity / RBAC / API conventions   ─── requires 1
    │
    ├── Phase 3: Sensor & Traffic Ingestion   ─── requires 2  ┐
    └── Phase 4: POS Ingestion & Conversion    ─── requires 2  ┘ (3 & 4 parallel)
             │
             ├── Phase 5: Forecasting & Scheduling   ─── requires 3 + 4
             ├── Phase 6: Basket Analysis            ─── requires 4   (6 parallel with 5)
             │
             ├── Phase 7: Overlays / Anomaly / Alerts ─── requires 5 (+ overlays feed 5)
             └── Phase 8: Benchmarking & dbt Marts    ─── requires 4 (5 for staffing gap)  (7 & 8 parallel)
                      │
                 Phase 9: NLQ & MCP            ─── requires 4–8 (queries over all metrics)
                      │
                 Phase 10: Dashboards & Heatmaps ─── requires 3–9 (consumes all endpoints)
                      │
    ┌─────────────────┴─────────────────┐
Phase 11: Privacy & Hardening        Phase 12: Packaging & Deployment
(both can run concurrently once 1–10 are functional; both required for release)
```

**Parallelism opportunities**
- Phases 3 and 4 can be built concurrently after Phase 2.
- Phases 5 and 6 can be built concurrently after Phase 4 (Phase 5 also needs Phase 3).
- Phases 7 and 8 can be built concurrently after Phase 5/4.
- Phases 11 and 12 can be built concurrently after the functional product (1–10) exists.

---

## Definition of Done (per phase)

Every phase must satisfy all of the following before it is considered complete:

1. All tasks in the phase implemented.
2. All unit and integration tests for the phase pass (`pytest`), including mocked-external and testcontainers-backed cases.
3. Ruff lint + format pass with no violations.
4. mypy (strict) passes.
5. Docker image builds and the affected services start cleanly.
6. The phase's feature works end-to-end against a real PostgreSQL + Redis (testcontainers or compose).
7. New configuration options are documented in `.env.example`.
8. New API endpoints appear in the auto-generated OpenAPI 3.1 spec with correct security schemes.
9. Database changes ship as a reversible Alembic migration (including partition DDL where relevant).
10. Multi-tenant + store-scope (BOLA) guards verified for any new object-scoped endpoint.
11. Any new external integration is mocked in CI and documented for real-credential use.
