# Data Model Suggestion 3: Hybrid Relational + JSONB

> Project: Supply Chain Analytics · Created: 2026-05-11

## Philosophy

This model keeps core, frequently-queried fields as strongly-typed relational columns while storing variable, jurisdiction-specific, and customer-configurable data in JSONB columns. The key insight is that a supply chain analytics platform serving mid-market manufacturers across different industries and geographies will encounter enormous variability in what metadata matters: a pharmaceutical company needs batch/lot tracking and FDA compliance fields; a food manufacturer needs temperature monitoring and FSMA traceability; an electronics distributor needs tariff classification codes and RoHS compliance flags. Rather than creating hundreds of nullable columns or separate tables for every industry vertical, a `metadata JSONB` column on each core entity absorbs this variability.

This is the approach used by modern SaaS platforms like Shopify (product metafields), Stripe (metadata on every object), and HubSpot (custom properties). PostgreSQL's JSONB support — with GIN indexing, containment operators (`@>`), and jsonpath queries — makes this pattern performant at scale. The hybrid approach gives you the best of both worlds: relational integrity for the core supply chain model (products, locations, forecasts, orders) and schema-on-write flexibility for everything else.

This is the fastest path to MVP because the core table count is low (~18 tables), schema changes for new customer requirements often don't need DDL migrations, and the platform can ship vertical-specific features (pharma compliance, food safety, electronics tariffs) through configuration rather than code.

**Best for:** Rapid MVP development targeting diverse mid-market verticals where customer-specific fields, multi-jurisdiction compliance requirements, and fast iteration are priorities.

**Trade-offs:**
- (+) Low table count (~18); fast to build and understand
- (+) Customer-specific and industry-specific fields without schema changes
- (+) JSONB fields are indexed and queryable with PostgreSQL GIN indexes
- (+) Natural fit for multi-region deployments where jurisdiction-specific fields vary
- (+) Fastest path to production MVP
- (-) JSONB fields lack referential integrity; application must enforce structure
- (-) BI tools may struggle with nested JSON; requires extraction for reporting
- (-) No database-enforced type checking on JSONB field contents
- (-) Risk of JSONB becoming a "junk drawer" without disciplined schema documentation
- (-) JSONB columns consume more storage than equivalent typed columns

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| SCOR Digital Standard | Core KPI columns are named after SCOR metrics; additional SCOR metrics stored in `metadata` JSONB |
| GS1 GTIN/GLN | Typed columns on product/location; additional GS1 identifiers (SSCC, GRAI) in metadata |
| ISO 3166-1/2 | `country_code` and `subdivision_code` as typed columns |
| ISO 4217 | `currency_code` as typed CHAR(3) column |
| ISO 31000 | Risk categories stored as JSONB structure within supplier risk assessments |
| CSDDD / LkSG | ESG compliance fields stored in supplier `compliance_data` JSONB — structure varies by jurisdiction |
| EPCIS 2.0 | Traceability events stored with `epcis_data` JSONB containing What/When/Where/Why/How dimensions |
| EDI X12 / EDIFACT | EDI transaction metadata stored in `edi_data` JSONB on purchase orders |

---

## Core Tables

### Tenancy

```sql
CREATE TABLE tenant (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    plan_tier       VARCHAR(50) NOT NULL DEFAULT 'free',
    industry        VARCHAR(100),              -- manufacturing, food_beverage, pharma, electronics, retail
    -- Industry-specific configuration stored here:
    -- {"default_currency": "EUR", "fiscal_year_start_month": 4, 
    --  "compliance_frameworks": ["csddd", "lksg"], 
    --  "custom_fields": {"product": [{"name": "hs_code", "type": "string", "label": "HS Tariff Code"}]}}
    settings        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE app_user (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    email           VARCHAR(255) NOT NULL,
    display_name    VARCHAR(255),
    role            VARCHAR(50) NOT NULL DEFAULT 'planner',
    preferences     JSONB NOT NULL DEFAULT '{}',  -- UI preferences, dashboard layout, notification settings
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, email)
);

-- RLS applied to all tenant-scoped tables
ALTER TABLE app_user ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON app_user
    USING (tenant_id = current_setting('app.current_tenant_id')::UUID);
```

### Product

```sql
CREATE TABLE product (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    sku             VARCHAR(100) NOT NULL,
    gtin            VARCHAR(14),
    name            VARCHAR(500) NOT NULL,
    category        VARCHAR(500),              -- flat category string, not normalized
    unit_of_measure VARCHAR(20) NOT NULL DEFAULT 'EA',
    unit_cost       NUMERIC(15, 4),
    currency_code   CHAR(3) DEFAULT 'USD',
    lead_time_days  INTEGER,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    -- Variable fields per industry/customer:
    -- Pharma: {"ndc": "1234-5678-90", "dea_schedule": "II", "lot_tracking": true, "cold_chain": true}
    -- Food:   {"shelf_life_days": 14, "temperature_class": "chilled", "allergens": ["gluten", "dairy"]}
    -- Electronics: {"hs_code": "8542.31", "rohs_compliant": true, "eccn": "3A001"}
    -- General: {"weight_kg": 0.5, "dimensions": {"l": 10, "w": 5, "h": 3, "unit": "cm"}}
    metadata        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, sku)
);

CREATE INDEX idx_product_tenant ON product(tenant_id);
CREATE INDEX idx_product_gtin ON product(gtin) WHERE gtin IS NOT NULL;
CREATE INDEX idx_product_category ON product(tenant_id, category);
-- GIN index for JSONB queries like: WHERE metadata @> '{"cold_chain": true}'
CREATE INDEX idx_product_metadata ON product USING GIN (metadata);
```

### Location

```sql
CREATE TABLE location (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    code            VARCHAR(100) NOT NULL,
    gln             VARCHAR(13),
    name            VARCHAR(500) NOT NULL,
    location_type   VARCHAR(50) NOT NULL,
    country_code    CHAR(2) NOT NULL,
    subdivision_code VARCHAR(6),
    timezone        VARCHAR(50) DEFAULT 'UTC',
    latitude        NUMERIC(10, 7),
    longitude       NUMERIC(10, 7),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    -- Variable fields:
    -- {"address": {"line1": "...", "city": "...", "postal_code": "..."},
    --  "capabilities": ["cold_storage", "hazmat", "cross_dock"],
    --  "operating_hours": {"mon_fri": "06:00-22:00", "sat": "08:00-14:00"},
    --  "certifications": ["iso_9001", "iso_14001", "brc_food_safety"]}
    metadata        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, code)
);

CREATE INDEX idx_location_tenant_type ON location(tenant_id, location_type);
CREATE INDEX idx_location_metadata ON location USING GIN (metadata);
```

### Supplier

```sql
CREATE TABLE supplier (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    code            VARCHAR(100) NOT NULL,
    name            VARCHAR(500) NOT NULL,
    country_code    CHAR(2),
    risk_tier       VARCHAR(20) DEFAULT 'unrated',
    is_active       BOOLEAN NOT NULL DEFAULT true,
    -- Structured data:
    -- {"lei": "5493001KJTIIGC8Y1R12",  -- ISO 17442
    --  "duns": "123456789",
    --  "contacts": [{"name": "...", "email": "...", "role": "primary"}],
    --  "payment_terms": {"net_days": 30, "currency": "EUR", "early_discount": "2/10"},
    --  "products_supplied": ["electronics", "sensors"]}
    metadata        JSONB NOT NULL DEFAULT '{}',
    -- CSDDD/LkSG compliance data — varies significantly by jurisdiction:
    -- {"lksg": {"risk_analysis_date": "2026-01-15", "tier": "direct", "human_rights_risk": "low",
    --           "environmental_risk": "medium", "corrective_actions": []},
    --  "csddd": {"due_diligence_date": "2026-03-01", "adverse_impacts_identified": false},
    --  "iso_certifications": ["iso_9001:2023-12-31", "iso_14001:2024-06-30"]}
    compliance_data JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, code)
);

CREATE INDEX idx_supplier_tenant ON supplier(tenant_id);
CREATE INDEX idx_supplier_risk ON supplier(tenant_id, risk_tier);
CREATE INDEX idx_supplier_compliance ON supplier USING GIN (compliance_data);
```

---

## Planning Tables

### Demand Forecast

```sql
CREATE TABLE demand_forecast (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    product_id      UUID NOT NULL REFERENCES product(id),
    location_id     UUID NOT NULL REFERENCES location(id),
    forecast_date   DATE NOT NULL,
    bucket_type     VARCHAR(10) NOT NULL DEFAULT 'weekly',
    quantity        NUMERIC(15, 2) NOT NULL,
    confidence_lower NUMERIC(15, 2),
    confidence_upper NUMERIC(15, 2),
    generation_id   UUID NOT NULL,
    is_overridden   BOOLEAN NOT NULL DEFAULT false,
    override_qty    NUMERIC(15, 2),
    override_reason TEXT,
    override_by     UUID REFERENCES app_user(id),
    -- Model details, SHAP values, external signals used:
    -- {"model": "tft_v2.3", "mape": 0.078, 
    --  "top_drivers": [{"feature": "promo_calendar", "shap": 0.23}, {"feature": "weather_temp", "shap": 0.15}],
    --  "external_signals": {"weather": true, "social_trends": false, "macro_gdp": true}}
    model_details   JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_forecast_lookup ON demand_forecast(tenant_id, product_id, location_id, forecast_date);
CREATE INDEX idx_forecast_generation ON demand_forecast(generation_id);
```

### Demand History

```sql
CREATE TABLE demand_history (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    product_id      UUID NOT NULL REFERENCES product(id),
    location_id     UUID NOT NULL REFERENCES location(id),
    demand_date     DATE NOT NULL,
    quantity        NUMERIC(15, 2) NOT NULL,
    revenue         NUMERIC(15, 2),
    source          VARCHAR(50),
    -- Source-specific fields:
    -- POS: {"register_id": "R001", "transaction_id": "TXN-123", "discount_pct": 10}
    -- EDI: {"edi_type": "850", "partner_id": "GLN-123", "interchange_id": "ISA-001"}
    -- ERP: {"erp_doc_number": "SO-2026-001", "erp_system": "sap_b1"}
    source_data     JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_demand_history ON demand_history(tenant_id, product_id, location_id, demand_date);
```

### Inventory

```sql
CREATE TABLE inventory_level (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    product_id      UUID NOT NULL REFERENCES product(id),
    location_id     UUID NOT NULL REFERENCES location(id),
    on_hand_qty     NUMERIC(15, 2) NOT NULL DEFAULT 0,
    allocated_qty   NUMERIC(15, 2) NOT NULL DEFAULT 0,
    in_transit_qty  NUMERIC(15, 2) NOT NULL DEFAULT 0,
    -- Batch/lot details (when applicable):
    -- {"batches": [
    --   {"batch_number": "LOT-2026-001", "quantity": 200, "expiry_date": "2026-08-15", "received_date": "2026-05-01"},
    --   {"batch_number": "LOT-2026-002", "quantity": 150, "expiry_date": "2026-09-30", "received_date": "2026-05-10"}
    -- ]}
    batch_details   JSONB NOT NULL DEFAULT '{}',
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, product_id, location_id)
);

CREATE INDEX idx_inventory_lookup ON inventory_level(tenant_id, product_id, location_id);

CREATE TABLE inventory_policy (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    product_id      UUID NOT NULL REFERENCES product(id),
    location_id     UUID NOT NULL REFERENCES location(id),
    safety_stock    NUMERIC(15, 2) NOT NULL,
    reorder_point   NUMERIC(15, 2) NOT NULL,
    reorder_qty     NUMERIC(15, 2) NOT NULL,
    service_level_target NUMERIC(5, 4) DEFAULT 0.95,
    calculation_method VARCHAR(50) DEFAULT 'statistical',
    -- Policy parameters vary by method:
    -- Statistical: {"z_score": 1.65, "demand_std_dev": 45.2, "lead_time_std_dev": 2.1}
    -- ML Probabilistic: {"model": "quantile_regression", "quantile": 0.95, "lookback_days": 365}
    -- Fixed: {"min_stock": 100, "max_stock": 500, "review_period_days": 7}
    parameters      JSONB NOT NULL DEFAULT '{}',
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, product_id, location_id)
);
```

### Purchase Orders

```sql
CREATE TABLE purchase_order (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    po_number       VARCHAR(100) NOT NULL,
    supplier_id     UUID NOT NULL REFERENCES supplier(id),
    ship_to_location_id UUID NOT NULL REFERENCES location(id),
    status          VARCHAR(30) NOT NULL DEFAULT 'draft',
    order_date      DATE NOT NULL,
    expected_delivery_date DATE,
    actual_delivery_date DATE,
    total_amount    NUMERIC(15, 2),
    currency_code   CHAR(3) DEFAULT 'USD',
    -- EDI and logistics details:
    -- {"edi_type": "x12_850", "interchange_id": "ISA-20260511-001",
    --  "shipping": {"carrier": "Maersk", "tracking": "MAEU1234567", "sscc": "00340123456789012345",
    --               "incoterms": "FOB", "port_of_loading": "CNSHA", "port_of_discharge": "USLAX"},
    --  "customs": {"hs_codes": ["8542.31"], "duty_rate_pct": 2.5, "country_of_origin": "CN"}}
    logistics_data  JSONB NOT NULL DEFAULT '{}',
    created_by      UUID REFERENCES app_user(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, po_number)
);

CREATE INDEX idx_po_supplier ON purchase_order(tenant_id, supplier_id);
CREATE INDEX idx_po_status ON purchase_order(tenant_id, status);

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
    -- Line-level details:
    -- {"batch_number": "LOT-2026-001", "quality_inspection": {"result": "pass", "inspector": "uuid"},
    --  "customs_clearance": {"cleared": true, "clearance_date": "2026-06-15"}}
    line_data       JSONB NOT NULL DEFAULT '{}',
    UNIQUE(tenant_id, purchase_order_id, line_number)
);
```

---

## Analytics Tables

### Supplier Risk

```sql
CREATE TABLE supplier_risk_assessment (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    supplier_id     UUID NOT NULL REFERENCES supplier(id),
    assessment_date DATE NOT NULL,
    overall_score   NUMERIC(5, 2) NOT NULL,
    risk_tier       VARCHAR(20) NOT NULL,
    -- Detailed scores and evidence — structure varies by methodology:
    -- {"financial": {"score": 85, "source": "d_and_b", "rating": "A2"},
    --  "operational": {"score": 70, "otif_90d": 0.92, "defect_rate": 0.005},
    --  "geopolitical": {"score": 55, "country_risk": "medium", "sanctions_check": "clear"},
    --  "esg": {"score": 80, "source": "sustainalytics", "carbon_intensity": "low"},
    --  "concentration": {"score": 60, "pct_of_spend": 0.35, "alternative_suppliers": 2},
    --  "llm_signals": [
    --    {"source": "reuters", "date": "2026-05-10", "headline": "...", "sentiment": -0.3},
    --    {"source": "annual_report", "date": "2026-03-01", "summary": "..."}
    --  ]}
    scores          JSONB NOT NULL,
    created_by      UUID REFERENCES app_user(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_risk_supplier ON supplier_risk_assessment(tenant_id, supplier_id, assessment_date);
CREATE INDEX idx_risk_scores ON supplier_risk_assessment USING GIN (scores);
```

### KPI Snapshots

```sql
CREATE TABLE kpi_snapshot (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    period_start    DATE NOT NULL,
    period_end      DATE NOT NULL,
    scope_type      VARCHAR(20) NOT NULL DEFAULT 'company',  -- company, location, product, supplier
    scope_id        UUID,                      -- NULL for company-wide
    -- Core SCOR metrics as typed columns for fast queries:
    fill_rate       NUMERIC(5, 4),
    inventory_turns NUMERIC(8, 2),
    mape            NUMERIC(8, 4),
    wmape           NUMERIC(8, 4),
    stockout_rate   NUMERIC(5, 4),
    -- Extended metrics in JSONB — varies by scope_type and industry:
    -- {"perfect_order_fulfillment": 0.945, "order_cycle_time_days": 3.2,
    --  "cash_to_cash_days": 45, "inventory_days_of_supply": 28,
    --  "cost_to_serve": 12500.00, "forecast_bias": -0.02,
    --  "supplier_otif_avg": 0.91, "sustainability": {"scope3_emissions_kg": 1250}}
    extended_metrics JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_kpi_period ON kpi_snapshot(tenant_id, period_start);
CREATE INDEX idx_kpi_scope ON kpi_snapshot(tenant_id, scope_type, scope_id);
```

### Audit Log

```sql
CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    user_id         UUID,
    entity_type     VARCHAR(100) NOT NULL,
    entity_id       UUID NOT NULL,
    action          VARCHAR(20) NOT NULL,
    changes         JSONB,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_entity ON audit_log(tenant_id, entity_type, entity_id);
CREATE INDEX idx_audit_time ON audit_log(tenant_id, created_at);
```

### Data Import

```sql
CREATE TABLE data_import (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    source_type     VARCHAR(50) NOT NULL,
    entity_type     VARCHAR(100) NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'pending',
    row_count       INTEGER,
    success_count   INTEGER,
    error_count     INTEGER,
    -- Import details and error log:
    -- {"file_name": "demand_history_2026.csv", "file_size_bytes": 1234567,
    --  "column_mapping": {"Date": "demand_date", "SKU": "sku", "Qty": "quantity"},
    --  "errors": [{"row": 5, "field": "sku", "value": "UNKNOWN-001", "error": "Product not found"}]}
    import_data     JSONB NOT NULL DEFAULT '{}',
    created_by      UUID REFERENCES app_user(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    completed_at    TIMESTAMPTZ
);
```

---

## JSONB Query Examples

### Find all cold-chain products

```sql
SELECT id, sku, name, metadata->>'temperature_class' AS temp_class
FROM product
WHERE tenant_id = :tenant_id
  AND metadata @> '{"cold_chain": true}';
-- Uses GIN index on metadata
```

### Find suppliers with CSDDD compliance gaps

```sql
SELECT s.id, s.name, s.compliance_data->'csddd' AS csddd_data
FROM supplier s
WHERE s.tenant_id = :tenant_id
  AND s.compliance_data->'csddd'->>'adverse_impacts_identified' = 'true';
```

### Get forecast SHAP drivers for a product

```sql
SELECT forecast_date, quantity, 
       model_details->'top_drivers' AS shap_drivers,
       model_details->>'model' AS model_name
FROM demand_forecast
WHERE tenant_id = :tenant_id
  AND product_id = :product_id
  AND generation_id = :latest_generation_id
ORDER BY forecast_date;
```

### Find POs with customs clearance issues

```sql
SELECT po.po_number, pol.line_number, pol.line_data->'customs_clearance' AS customs
FROM purchase_order po
JOIN purchase_order_line pol ON pol.purchase_order_id = po.id
WHERE po.tenant_id = :tenant_id
  AND pol.line_data->'customs_clearance'->>'cleared' = 'false';
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Tenancy & Auth | 2 | tenant, app_user |
| Master Data | 3 | product, location, supplier |
| Planning | 4 | demand_forecast, demand_history, inventory_level, inventory_policy |
| Procurement | 2 | purchase_order, purchase_order_line |
| Analytics | 2 | supplier_risk_assessment, kpi_snapshot |
| System | 2 | audit_log, data_import |
| **Total** | **15** | JSONB absorbs what would be 8-10 additional tables in Model 1 |

---

## Key Design Decisions

1. **Metadata JSONB on every master data entity** — rather than creating `product_attribute`, `location_capability`, and `supplier_certification` junction tables, a single `metadata` JSONB column on each entity absorbs variable fields. This eliminates ~8 tables from the normalized model.

2. **Separate `compliance_data` on supplier** — regulatory compliance data (CSDDD, LkSG, ISO certifications) is kept in its own JSONB column rather than merged into `metadata`, because (a) it will be queried independently for compliance reporting and (b) it changes on a different cadence than operational metadata.

3. **Forecast override fields inline** — rather than a separate `forecast_override` table, override data (`is_overridden`, `override_qty`, `override_reason`, `override_by`) lives on the `demand_forecast` row itself. This simplifies the most common query ("show me the effective forecast") from a JOIN to a single table scan.

4. **Batch details as JSONB array** — for products requiring lot/batch tracking, `inventory_level.batch_details` stores an array of batch objects. Companies without batch tracking simply leave this as `{}`. This avoids a separate `inventory_batch` table that would be empty for most mid-market customers.

5. **Logistics data on purchase orders** — shipping, customs, and EDI details are stored in `logistics_data` JSONB because the structure varies dramatically by trade lane, carrier, and incoterms. A US domestic shipment has no customs data; an international ocean freight shipment has port codes, container numbers, HS codes, and duty rates.

6. **Flat category string** — product categories are stored as a flat string ("Electronics/Sensors/Temperature") rather than a normalized category hierarchy. For a mid-market analytics platform, the simplicity of a string column with a text index outweighs the query flexibility of a normalized tree. Categories can be parsed by the application layer if hierarchical aggregation is needed.

7. **Core SCOR metrics as typed columns, extended metrics in JSONB** — the five most commonly queried KPIs (fill_rate, inventory_turns, mape, wmape, stockout_rate) are typed columns for fast filtering and aggregation. All other metrics live in `extended_metrics` JSONB, allowing customers to define custom KPIs without schema changes.

8. **GIN indexes on all JSONB columns** — PostgreSQL GIN indexes support the `@>` containment operator, making queries like `WHERE metadata @> '{"cold_chain": true}'` efficient. For frequently-queried JSONB paths, expression indexes can be added: `CREATE INDEX idx_product_hs ON product((metadata->>'hs_code'))`.

9. **Tenant settings define custom fields** — the `tenant.settings` JSONB includes a `custom_fields` definition that the UI layer reads to render dynamic forms. This enables customers to define their own fields ("HS Tariff Code", "Allergen Class") without any backend changes.

10. **No separate supplier_product junction** — instead of a normalized junction table mapping suppliers to products, supplier product catalogs are stored in the supplier's `metadata` or referenced inline on purchase order lines. This reduces table count and JOIN complexity for the common case where the supplier-product relationship is captured implicitly through order history.
