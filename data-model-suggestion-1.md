# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: Supply Chain Analytics · Created: 2026-05-11

## Philosophy

This model follows a classic third-normal-form (3NF) relational design where every domain concept — products, locations, suppliers, forecasts, inventory positions, purchase orders, and performance metrics — has its own dedicated table with explicit foreign key relationships. The schema is aligned to the SCOR Digital Standard process domains (Plan, Source, Make, Deliver, Return, Enable) and uses GS1 identifiers (GTIN, GLN) as first-class columns for interoperability with trading partners.

The design philosophy is that supply chain data integrity is non-negotiable. A demand forecast should always reference a valid product and location. A purchase order line should always trace back to a supplier and a product. Inventory levels should be constrained to known locations. By encoding these relationships as foreign keys, the database itself enforces data quality — preventing orphaned records, phantom suppliers, or forecasts for non-existent SKUs.

This is the approach used by frePPLe (the only production open-source supply chain planning platform) and by SAP IBP's underlying OData data model. It is the natural fit for teams who value SQL queryability, tooling ecosystem compatibility, and schema-enforced consistency.

**Best for:** Teams building a production-grade supply chain planning platform where data integrity, standard SQL tooling, and SCOR metric alignment are priorities.

**Trade-offs:**
- (+) Strong referential integrity; the database enforces business rules
- (+) Standard SQL queries; compatible with any BI tool (Metabase, Superset, Tableau)
- (+) Clear, well-documented schema; easy for new developers to understand
- (+) Natural alignment with SCOR metrics and GS1 identifiers
- (-) High table count (~35-40 tables); schema migrations require coordination
- (-) Schema changes for new entity types require DDL + migration
- (-) Many-to-many relationships require junction tables, increasing join complexity
- (-) Less flexible for jurisdiction-specific or customer-specific custom fields

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| SCOR Digital Standard (ASCM) | KPI table metrics align to SCOR Level 1/2 metric definitions (perfect order fulfilment, supply chain cycle time, inventory turns) |
| GS1 GTIN | `product.gtin` column stores Global Trade Item Number for product identification |
| GS1 GLN | `location.gln` column stores Global Location Number for facility identification |
| GS1 SSCC | `shipment.sscc` column stores Serial Shipping Container Code for shipment tracking |
| ISO 3166-1/2 | `location.country_code` and `location.subdivision_code` use ISO 3166 codes |
| ISO 4217 | `currency_code` columns use ISO 4217 three-letter currency codes |
| ISO 8601 | All temporal columns use `TIMESTAMPTZ` with ISO 8601 formatting |
| ISO 31000 | Supplier risk scoring table structure follows ISO 31000 risk assessment categories |
| EDI X12 850/856 | Purchase order and shipment entities map to X12 850 (PO) and 856 (ASN) segment structures |
| EPCIS 2.0 | Event table structure captures What/When/Where/Why/How dimensions from EPCIS |

---

## Core Infrastructure Tables

### Tenancy & Authentication

```sql
CREATE TABLE tenant (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    plan_tier       VARCHAR(50) NOT NULL DEFAULT 'free',  -- free, pro, enterprise
    settings        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE app_user (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    email           VARCHAR(255) NOT NULL,
    display_name    VARCHAR(255),
    role            VARCHAR(50) NOT NULL DEFAULT 'planner',  -- admin, planner, viewer, api
    password_hash   VARCHAR(255),
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, email)
);

CREATE INDEX idx_app_user_tenant ON app_user(tenant_id);
```

### Row-Level Security

```sql
ALTER TABLE tenant ENABLE ROW LEVEL SECURITY;
ALTER TABLE app_user ENABLE ROW LEVEL SECURITY;

-- Pattern applied to all tenant-scoped tables:
CREATE POLICY tenant_isolation ON app_user
    USING (tenant_id = current_setting('app.current_tenant_id')::UUID);
```

---

## Master Data Tables

### Product

```sql
CREATE TABLE product (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    sku             VARCHAR(100) NOT NULL,
    gtin            VARCHAR(14),               -- GS1 Global Trade Item Number
    name            VARCHAR(500) NOT NULL,
    description     TEXT,
    category_id     UUID REFERENCES product_category(id),
    unit_of_measure VARCHAR(20) NOT NULL DEFAULT 'EA',  -- EA, KG, L, CS, PL
    unit_cost       NUMERIC(15, 4),
    currency_code   CHAR(3) DEFAULT 'USD',     -- ISO 4217
    lead_time_days  INTEGER,
    shelf_life_days INTEGER,
    weight_kg       NUMERIC(10, 3),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, sku)
);

CREATE INDEX idx_product_tenant ON product(tenant_id);
CREATE INDEX idx_product_gtin ON product(gtin) WHERE gtin IS NOT NULL;
CREATE INDEX idx_product_category ON product(category_id);

CREATE TABLE product_category (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            VARCHAR(255) NOT NULL,
    parent_id       UUID REFERENCES product_category(id),
    level           INTEGER NOT NULL DEFAULT 0,
    path            TEXT,  -- materialized path: "Electronics/Sensors/Temperature"
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### Location

```sql
CREATE TABLE location (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    code            VARCHAR(100) NOT NULL,
    gln             VARCHAR(13),               -- GS1 Global Location Number
    name            VARCHAR(500) NOT NULL,
    location_type   VARCHAR(50) NOT NULL,       -- warehouse, factory, distribution_center, store, supplier_site
    address_line1   VARCHAR(500),
    address_line2   VARCHAR(500),
    city            VARCHAR(255),
    state_province  VARCHAR(255),
    postal_code     VARCHAR(20),
    country_code    CHAR(2) NOT NULL,           -- ISO 3166-1 alpha-2
    subdivision_code VARCHAR(6),                -- ISO 3166-2
    latitude        NUMERIC(10, 7),
    longitude       NUMERIC(10, 7),
    timezone        VARCHAR(50) DEFAULT 'UTC',
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, code)
);

CREATE INDEX idx_location_tenant ON location(tenant_id);
CREATE INDEX idx_location_gln ON location(gln) WHERE gln IS NOT NULL;
CREATE INDEX idx_location_type ON location(tenant_id, location_type);
```

### Supplier

```sql
CREATE TABLE supplier (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    code            VARCHAR(100) NOT NULL,
    name            VARCHAR(500) NOT NULL,
    legal_entity_id VARCHAR(20),               -- LEI (ISO 17442) if available
    duns_number     VARCHAR(9),                -- D-U-N-S Number
    primary_contact_name  VARCHAR(255),
    primary_contact_email VARCHAR(255),
    payment_terms_days    INTEGER DEFAULT 30,
    currency_code   CHAR(3) DEFAULT 'USD',     -- ISO 4217
    country_code    CHAR(2),                   -- ISO 3166-1
    risk_tier       VARCHAR(20) DEFAULT 'unrated',  -- low, medium, high, critical, unrated
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, code)
);

CREATE INDEX idx_supplier_tenant ON supplier(tenant_id);
CREATE INDEX idx_supplier_risk ON supplier(tenant_id, risk_tier);

-- Junction: which supplier can supply which product from which location
CREATE TABLE supplier_product (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    supplier_id     UUID NOT NULL REFERENCES supplier(id),
    product_id      UUID NOT NULL REFERENCES product(id),
    supplier_sku    VARCHAR(100),
    unit_cost       NUMERIC(15, 4),
    currency_code   CHAR(3) DEFAULT 'USD',
    min_order_qty   NUMERIC(15, 2),
    lead_time_days  INTEGER NOT NULL,
    is_preferred    BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, supplier_id, product_id)
);
```

### Customer

```sql
CREATE TABLE customer (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    code            VARCHAR(100) NOT NULL,
    name            VARCHAR(500) NOT NULL,
    customer_type   VARCHAR(50),               -- retail, wholesale, distributor, internal
    country_code    CHAR(2),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, code)
);
```

---

## Demand Planning Tables

### Demand Forecast

```sql
CREATE TABLE demand_forecast (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    product_id      UUID NOT NULL REFERENCES product(id),
    location_id     UUID NOT NULL REFERENCES location(id),
    customer_id     UUID REFERENCES customer(id),
    forecast_date   DATE NOT NULL,             -- the date being forecasted
    bucket_type     VARCHAR(10) NOT NULL,       -- daily, weekly, monthly
    quantity        NUMERIC(15, 2) NOT NULL,
    unit_of_measure VARCHAR(20) NOT NULL DEFAULT 'EA',
    forecast_model  VARCHAR(100),              -- arima, lightgbm, tft, ensemble
    confidence_lower NUMERIC(15, 2),           -- lower bound (e.g. P10)
    confidence_upper NUMERIC(15, 2),           -- upper bound (e.g. P90)
    mape            NUMERIC(8, 4),             -- mean absolute percentage error
    generation_id   UUID NOT NULL,             -- links all rows from one forecast run
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_forecast_lookup ON demand_forecast(tenant_id, product_id, location_id, forecast_date);
CREATE INDEX idx_forecast_generation ON demand_forecast(generation_id);
CREATE INDEX idx_forecast_date_range ON demand_forecast(tenant_id, forecast_date);

CREATE TABLE forecast_generation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    model_type      VARCHAR(100) NOT NULL,     -- lightgbm, temporal_fusion_transformer, ensemble
    model_version   VARCHAR(50),
    parameters      JSONB,                     -- hyperparameters used
    training_start  DATE,
    training_end    DATE,
    overall_mape    NUMERIC(8, 4),
    overall_wmape   NUMERIC(8, 4),
    status          VARCHAR(20) NOT NULL DEFAULT 'running',  -- running, completed, failed
    created_by      UUID REFERENCES app_user(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    completed_at    TIMESTAMPTZ
);
```

### Demand History (Actuals)

```sql
CREATE TABLE demand_history (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    product_id      UUID NOT NULL REFERENCES product(id),
    location_id     UUID NOT NULL REFERENCES location(id),
    customer_id     UUID REFERENCES customer(id),
    demand_date     DATE NOT NULL,
    quantity        NUMERIC(15, 2) NOT NULL,
    unit_of_measure VARCHAR(20) NOT NULL DEFAULT 'EA',
    revenue         NUMERIC(15, 2),
    source          VARCHAR(50),               -- pos, erp, edi, manual
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_demand_history_lookup ON demand_history(tenant_id, product_id, location_id, demand_date);
CREATE INDEX idx_demand_history_date ON demand_history(tenant_id, demand_date);
```

### Forecast Override (Planner Adjustments)

```sql
CREATE TABLE forecast_override (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    forecast_id     UUID NOT NULL REFERENCES demand_forecast(id),
    original_qty    NUMERIC(15, 2) NOT NULL,
    override_qty    NUMERIC(15, 2) NOT NULL,
    reason          TEXT,
    override_type   VARCHAR(50),               -- promotion, market_intelligence, executive_override
    approved_by     UUID REFERENCES app_user(id),
    approved_at     TIMESTAMPTZ,
    status          VARCHAR(20) NOT NULL DEFAULT 'pending',  -- pending, approved, rejected
    created_by      UUID NOT NULL REFERENCES app_user(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Inventory Management Tables

```sql
CREATE TABLE inventory_level (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    product_id      UUID NOT NULL REFERENCES product(id),
    location_id     UUID NOT NULL REFERENCES location(id),
    on_hand_qty     NUMERIC(15, 2) NOT NULL DEFAULT 0,
    allocated_qty   NUMERIC(15, 2) NOT NULL DEFAULT 0,
    in_transit_qty  NUMERIC(15, 2) NOT NULL DEFAULT 0,
    available_qty   NUMERIC(15, 2) GENERATED ALWAYS AS (on_hand_qty - allocated_qty) STORED,
    batch_number    VARCHAR(100),
    expiry_date     DATE,
    last_counted_at TIMESTAMPTZ,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, product_id, location_id, COALESCE(batch_number, ''))
);

CREATE INDEX idx_inventory_lookup ON inventory_level(tenant_id, product_id, location_id);
CREATE INDEX idx_inventory_expiry ON inventory_level(expiry_date) WHERE expiry_date IS NOT NULL;

CREATE TABLE inventory_policy (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    product_id      UUID NOT NULL REFERENCES product(id),
    location_id     UUID NOT NULL REFERENCES location(id),
    safety_stock    NUMERIC(15, 2) NOT NULL,
    reorder_point   NUMERIC(15, 2) NOT NULL,
    reorder_qty     NUMERIC(15, 2) NOT NULL,
    max_stock       NUMERIC(15, 2),
    review_period_days INTEGER DEFAULT 7,
    service_level_target NUMERIC(5, 4) DEFAULT 0.95,  -- e.g. 0.95 = 95%
    calculation_method VARCHAR(50) DEFAULT 'statistical',  -- statistical, ml_probabilistic, fixed
    last_calculated_at TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, product_id, location_id)
);
```

---

## Procurement Tables

```sql
CREATE TABLE purchase_order (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    po_number       VARCHAR(100) NOT NULL,
    supplier_id     UUID NOT NULL REFERENCES supplier(id),
    ship_to_location_id UUID NOT NULL REFERENCES location(id),
    status          VARCHAR(30) NOT NULL DEFAULT 'draft',  -- draft, submitted, confirmed, shipped, received, cancelled
    order_date      DATE NOT NULL,
    expected_delivery_date DATE,
    actual_delivery_date DATE,
    total_amount    NUMERIC(15, 2),
    currency_code   CHAR(3) DEFAULT 'USD',
    notes           TEXT,
    edi_reference   VARCHAR(100),              -- X12 850 interchange control number
    created_by      UUID REFERENCES app_user(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, po_number)
);

CREATE INDEX idx_po_supplier ON purchase_order(tenant_id, supplier_id);
CREATE INDEX idx_po_status ON purchase_order(tenant_id, status);
CREATE INDEX idx_po_delivery ON purchase_order(tenant_id, expected_delivery_date);

CREATE TABLE purchase_order_line (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    purchase_order_id UUID NOT NULL REFERENCES purchase_order(id),
    line_number     INTEGER NOT NULL,
    product_id      UUID NOT NULL REFERENCES product(id),
    quantity        NUMERIC(15, 2) NOT NULL,
    unit_cost       NUMERIC(15, 4) NOT NULL,
    received_qty    NUMERIC(15, 2) DEFAULT 0,
    rejected_qty    NUMERIC(15, 2) DEFAULT 0,
    unit_of_measure VARCHAR(20) NOT NULL DEFAULT 'EA',
    UNIQUE(tenant_id, purchase_order_id, line_number)
);
```

---

## Supply Planning Tables

```sql
CREATE TABLE supply_plan (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    product_id      UUID NOT NULL REFERENCES product(id),
    location_id     UUID NOT NULL REFERENCES location(id),
    plan_date       DATE NOT NULL,
    bucket_type     VARCHAR(10) NOT NULL,       -- daily, weekly, monthly
    planned_receipts NUMERIC(15, 2) DEFAULT 0,
    planned_shipments NUMERIC(15, 2) DEFAULT 0,
    projected_on_hand NUMERIC(15, 2) DEFAULT 0,
    projected_available NUMERIC(15, 2) DEFAULT 0,
    generation_id   UUID NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_supply_plan_lookup ON supply_plan(tenant_id, product_id, location_id, plan_date);

CREATE TABLE replenishment_recommendation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    product_id      UUID NOT NULL REFERENCES product(id),
    location_id     UUID NOT NULL REFERENCES location(id),
    supplier_id     UUID REFERENCES supplier(id),
    recommended_qty NUMERIC(15, 2) NOT NULL,
    recommended_date DATE NOT NULL,
    urgency         VARCHAR(20) NOT NULL DEFAULT 'normal',  -- critical, urgent, normal
    reason          TEXT,
    status          VARCHAR(20) NOT NULL DEFAULT 'pending',  -- pending, approved, rejected, ordered
    approved_by     UUID REFERENCES app_user(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Supplier Performance & Risk Tables

```sql
CREATE TABLE supplier_performance (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    supplier_id     UUID NOT NULL REFERENCES supplier(id),
    period_start    DATE NOT NULL,
    period_end      DATE NOT NULL,
    on_time_delivery_rate  NUMERIC(5, 4),      -- 0.0000 to 1.0000
    in_full_rate           NUMERIC(5, 4),
    otif_rate              NUMERIC(5, 4),       -- on-time in-full combined
    avg_lead_time_days     NUMERIC(8, 2),
    lead_time_std_dev      NUMERIC(8, 2),
    defect_rate            NUMERIC(5, 4),
    total_orders           INTEGER,
    total_lines            INTEGER,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_supplier_perf ON supplier_performance(tenant_id, supplier_id, period_start);

CREATE TABLE supplier_risk_score (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    supplier_id     UUID NOT NULL REFERENCES supplier(id),
    assessment_date DATE NOT NULL,
    overall_score   NUMERIC(5, 2) NOT NULL,    -- 0-100 composite
    financial_score NUMERIC(5, 2),              -- ISO 31000 risk category
    operational_score NUMERIC(5, 2),
    geopolitical_score NUMERIC(5, 2),
    esg_score       NUMERIC(5, 2),             -- for CSDDD/LkSG compliance
    concentration_score NUMERIC(5, 2),          -- geographic/product concentration risk
    methodology     VARCHAR(50),                -- structured, llm_enhanced, composite
    evidence_summary TEXT,
    created_by      UUID REFERENCES app_user(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_risk_score ON supplier_risk_score(tenant_id, supplier_id, assessment_date);
```

---

## SCOR-Aligned KPI Tables

```sql
CREATE TABLE kpi_snapshot (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    location_id     UUID REFERENCES location(id),  -- NULL = company-wide
    product_id      UUID REFERENCES product(id),   -- NULL = all products
    period_start    DATE NOT NULL,
    period_end      DATE NOT NULL,
    -- SCOR Reliability metrics
    perfect_order_fulfillment   NUMERIC(5, 4),     -- RL.1.1
    order_fulfillment_cycle_time NUMERIC(8, 2),    -- RS.1.1 (days)
    -- SCOR Responsiveness metrics
    fill_rate                   NUMERIC(5, 4),
    -- SCOR Agility metrics
    upside_supply_chain_flexibility INTEGER,        -- AG.1.1 (days)
    -- SCOR Cost metrics
    total_scm_cost              NUMERIC(15, 2),    -- CO.1.1
    cost_of_goods_sold          NUMERIC(15, 2),
    -- SCOR Asset Management metrics
    cash_to_cash_cycle_days     NUMERIC(8, 2),     -- AM.1.1
    inventory_days_of_supply    NUMERIC(8, 2),     -- AM.1.3
    inventory_turns             NUMERIC(8, 2),
    -- Forecast accuracy
    mape                        NUMERIC(8, 4),
    wmape                       NUMERIC(8, 4),
    forecast_bias               NUMERIC(8, 4),
    -- Inventory health
    stockout_rate               NUMERIC(5, 4),
    excess_inventory_value      NUMERIC(15, 2),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_kpi_period ON kpi_snapshot(tenant_id, period_start, period_end);
```

---

## Audit Trail

```sql
CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    user_id         UUID REFERENCES app_user(id),
    entity_type     VARCHAR(100) NOT NULL,     -- purchase_order, forecast_override, etc.
    entity_id       UUID NOT NULL,
    action          VARCHAR(20) NOT NULL,       -- create, update, delete, approve, reject
    changes         JSONB,                     -- {"field": {"old": x, "new": y}}
    ip_address      INET,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_entity ON audit_log(tenant_id, entity_type, entity_id);
CREATE INDEX idx_audit_time ON audit_log(tenant_id, created_at);
```

---

## Data Ingestion

```sql
CREATE TABLE data_import (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    source_type     VARCHAR(50) NOT NULL,       -- csv, api, edi_x12, edi_edifact, erp_sap, erp_oracle
    entity_type     VARCHAR(100) NOT NULL,      -- product, demand_history, purchase_order, inventory_level
    file_name       VARCHAR(500),
    row_count       INTEGER,
    success_count   INTEGER,
    error_count     INTEGER,
    errors          JSONB,                     -- [{row: 5, field: "sku", error: "not found"}]
    status          VARCHAR(20) NOT NULL DEFAULT 'pending',  -- pending, processing, completed, failed
    created_by      UUID REFERENCES app_user(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    completed_at    TIMESTAMPTZ
);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Tenancy & Auth | 2 | tenant, app_user |
| Master Data | 5 | product, product_category, location, supplier, customer |
| Supplier Relationships | 1 | supplier_product (junction) |
| Demand Planning | 4 | demand_forecast, forecast_generation, demand_history, forecast_override |
| Inventory | 2 | inventory_level, inventory_policy |
| Procurement | 2 | purchase_order, purchase_order_line |
| Supply Planning | 2 | supply_plan, replenishment_recommendation |
| Supplier Analytics | 2 | supplier_performance, supplier_risk_score |
| KPIs | 1 | kpi_snapshot (SCOR-aligned) |
| System | 2 | audit_log, data_import |
| **Total** | **23** | |

---

## Key Design Decisions

1. **UUID primary keys everywhere** — enables distributed ID generation without coordination, essential for multi-tenant SaaS where clients may import data from CSV with their own IDs.

2. **tenant_id on every table with RLS** — PostgreSQL row-level security provides defence-in-depth isolation. Even if application code has a bug, one tenant cannot see another's data.

3. **GS1 identifiers as optional columns** — GTIN and GLN are indexed but nullable, allowing companies without GS1 memberships to use internal SKU/location codes while enabling seamless trading-partner interoperability for those who do.

4. **Forecast generations as first-class entities** — each forecast run produces a `forecast_generation` row linking to all its `demand_forecast` rows. This enables comparing model versions, A/B testing forecasting algorithms, and auditing which model was active when a planning decision was made.

5. **Computed `available_qty`** — the `inventory_level.available_qty` column is a PostgreSQL generated column (`on_hand_qty - allocated_qty`), eliminating the risk of stale calculations.

6. **SCOR metric names in KPI table** — column names like `perfect_order_fulfillment` and `cash_to_cash_cycle_days` directly reference SCOR metric IDs (RL.1.1, AM.1.1), making it straightforward for enterprise customers to map platform KPIs to their existing SCOR benchmarking processes.

7. **Explicit purchase order status machine** — the `purchase_order.status` column follows the lifecycle (draft → submitted → confirmed → shipped → received → cancelled), which maps to EDI X12 transaction flow (850 → 855 → 856 → receiving).

8. **Supplier risk scores as time-series** — each `supplier_risk_score` row is dated, enabling trend analysis of supplier risk over time rather than storing only the current score.

9. **Audit log with JSONB diffs** — the `changes` column stores old/new value pairs as JSONB, providing a complete audit trail without requiring separate history tables for every entity.

10. **Materialized path for product categories** — the `path` column in `product_category` enables efficient hierarchical queries (e.g., "all products under Electronics") without recursive CTEs, while `parent_id` preserves the adjacency-list structure for tree manipulation.
