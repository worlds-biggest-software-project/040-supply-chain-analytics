# Data Model Suggestion 4: Graph-Relational Hybrid

> Project: Supply Chain Analytics · Created: 2026-05-11

## Philosophy

This model combines a relational PostgreSQL layer for operational CRUD (forecasts, inventory, purchase orders, KPIs) with a graph layer for modeling the complex, many-to-many, multi-tier relationships that are the defining challenge of supply chain analytics. The graph layer captures supplier networks, material flows, product-to-component bills of materials, location-to-location transport links, and risk propagation paths — relationships that are awkward to model with junction tables and expensive to traverse with recursive CTEs.

The graph can be implemented in two ways: (a) using a dedicated graph database like Neo4j alongside PostgreSQL, or (b) using PostgreSQL-native graph tables (`graph_node` / `graph_edge`) with `ltree` extensions and recursive CTEs. This proposal uses option (b) — a PostgreSQL-native graph layer — because it avoids the operational complexity of running two databases while still enabling the relationship-heavy queries that differentiate supply chain analytics from generic BI.

This approach mirrors o9 Solutions' architecture, which uses a graph-based AI platform to model supply chain relationships natively. Neo4j's supply chain use cases — supplier dependency mapping, risk propagation analysis, alternative supplier path-finding, and bill-of-materials explosion — are the exact queries that mid-market manufacturers struggle to perform with flat relational schemas. The graph layer makes these queries natural and efficient.

**Best for:** Platforms where supplier network analysis, multi-tier risk propagation, bill-of-materials explosion, and relationship-heavy queries (shortest path, impact analysis, concentration risk) are primary differentiators.

**Trade-offs:**
- (+) Natural modeling of multi-tier supplier networks and material flows
- (+) Efficient path-finding: "which suppliers are affected if Shanghai port closes?"
- (+) Bill-of-materials traversal without complex recursive CTEs
- (+) Risk propagation analysis: "if Supplier X fails, what products are impacted?"
- (+) Concentration risk detection: "are we over-dependent on a single sub-tier supplier?"
- (+) Graph queries compose naturally for complex cross-entity analysis
- (-) Graph query language (Cypher or recursive SQL) has a steeper learning curve
- (-) PostgreSQL-native graph requires careful index design for traversal performance
- (-) Two mental models (relational + graph) increase developer onboarding time
- (-) Graph layer adds complexity for simple CRUD operations that don't need traversal
- (-) Fewer off-the-shelf BI tools support graph data natively

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| SCOR Digital Standard | Graph edges typed by SCOR process domain (Source, Make, Deliver) to model process flows |
| GS1 GTIN/GLN/SSCC | Graph nodes for products, locations, and shipments carry GS1 identifiers as properties |
| ISO 31000 | Risk propagation through graph edges models the ISO 31000 risk pathway concept |
| ISO 22301 / ISO/TS 22318 | Supply chain continuity analysis uses graph traversal to identify critical path dependencies |
| ISO 28000 | Security management across the supply chain modeled as risk edges between supplier nodes |
| CSDDD / LkSG | Multi-tier supplier due diligence uses graph traversal to trace beyond direct (Tier 1) suppliers |
| UN/CEFACT SCRDM | Trade Party and Consignment entities from SCRDM map to graph nodes and relationship edges |
| EPCIS 2.0 | Product movement events create edges between location nodes, building the supply chain visibility graph |

---

## Graph Layer

### Node and Edge Tables

```sql
-- Generic graph node table: every entity that participates in supply chain relationships
CREATE TABLE graph_node (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    node_type       VARCHAR(50) NOT NULL,       -- supplier, product, location, component, material, customer
    entity_id       UUID NOT NULL,              -- FK to the relational table (product.id, supplier.id, etc.)
    label           VARCHAR(500) NOT NULL,       -- human-readable label for visualization
    properties      JSONB NOT NULL DEFAULT '{}', -- node-specific properties for graph queries
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, node_type, entity_id)
);

CREATE INDEX idx_gnode_tenant_type ON graph_node(tenant_id, node_type);
CREATE INDEX idx_gnode_entity ON graph_node(entity_id);
CREATE INDEX idx_gnode_properties ON graph_node USING GIN (properties);

-- Generic graph edge table: relationships between nodes
CREATE TABLE graph_edge (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    source_node_id  UUID NOT NULL REFERENCES graph_node(id),
    target_node_id  UUID NOT NULL REFERENCES graph_node(id),
    edge_type       VARCHAR(100) NOT NULL,      -- see Edge Type Catalog below
    properties      JSONB NOT NULL DEFAULT '{}', -- edge-specific properties (cost, lead time, capacity)
    weight          NUMERIC(10, 4) DEFAULT 1.0, -- for weighted path-finding algorithms
    is_active       BOOLEAN NOT NULL DEFAULT true,
    valid_from      TIMESTAMPTZ DEFAULT now(),
    valid_to        TIMESTAMPTZ,                -- NULL = currently active
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_gedge_source ON graph_edge(source_node_id, edge_type) WHERE is_active = true;
CREATE INDEX idx_gedge_target ON graph_edge(target_node_id, edge_type) WHERE is_active = true;
CREATE INDEX idx_gedge_tenant_type ON graph_edge(tenant_id, edge_type);
CREATE INDEX idx_gedge_properties ON graph_edge USING GIN (properties);
```

### Edge Type Catalog

| Edge Type | Source Node | Target Node | Description | Example Properties |
|-----------|-------------|-------------|-------------|-------------------|
| `SUPPLIES` | supplier | product | Supplier can supply this product | `{"lead_time_days": 14, "unit_cost": 12.50, "min_order_qty": 100, "is_preferred": true}` |
| `SOURCES_FROM` | location | supplier | Location sources from this supplier | `{"contract_id": "uuid", "annual_volume": 50000}` |
| `SHIPS_TO` | location | location | Transport link between locations | `{"transport_mode": "ocean", "transit_days": 21, "cost_per_unit": 0.85, "carrier": "Maersk"}` |
| `STORES_AT` | product | location | Product is stocked at this location | `{"safety_stock": 500, "reorder_point": 200}` |
| `COMPONENT_OF` | product | product | BOM relationship (component → parent) | `{"quantity_per": 4, "scrap_rate": 0.02, "bom_level": 2}` |
| `MANUFACTURED_AT` | product | location | Product is made at this location | `{"capacity_per_day": 1000, "setup_time_hours": 2}` |
| `SOLD_TO` | product | customer | Product is sold to this customer | `{"avg_monthly_volume": 500, "price": 25.00}` |
| `SUB_SUPPLIER_OF` | supplier | supplier | Tier 2+ supplier relationship | `{"tier": 2, "material_type": "raw_material", "criticality": "high"}` |
| `RISK_EXPOSURE` | supplier | supplier | Shared risk factor (geography, industry) | `{"risk_type": "geographic", "region": "East Asia", "correlation": 0.85}` |
| `ALTERNATIVE_TO` | supplier | supplier | Alternative/backup supplier for same products | `{"qualification_status": "approved", "cost_premium_pct": 15}` |

---

## Relational Layer (Operational Tables)

### Tenancy & Auth

```sql
CREATE TABLE tenant (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    plan_tier       VARCHAR(50) NOT NULL DEFAULT 'free',
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
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, email)
);
```

### Product, Location, Supplier (Master Data)

```sql
CREATE TABLE product (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    sku             VARCHAR(100) NOT NULL,
    gtin            VARCHAR(14),
    name            VARCHAR(500) NOT NULL,
    category        VARCHAR(500),
    unit_of_measure VARCHAR(20) NOT NULL DEFAULT 'EA',
    unit_cost       NUMERIC(15, 4),
    currency_code   CHAR(3) DEFAULT 'USD',
    lead_time_days  INTEGER,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    metadata        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, sku)
);

CREATE TABLE location (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    code            VARCHAR(100) NOT NULL,
    gln             VARCHAR(13),
    name            VARCHAR(500) NOT NULL,
    location_type   VARCHAR(50) NOT NULL,
    country_code    CHAR(2) NOT NULL,
    latitude        NUMERIC(10, 7),
    longitude       NUMERIC(10, 7),
    timezone        VARCHAR(50) DEFAULT 'UTC',
    is_active       BOOLEAN NOT NULL DEFAULT true,
    metadata        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, code)
);

CREATE TABLE supplier (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    code            VARCHAR(100) NOT NULL,
    name            VARCHAR(500) NOT NULL,
    country_code    CHAR(2),
    risk_tier       VARCHAR(20) DEFAULT 'unrated',
    is_active       BOOLEAN NOT NULL DEFAULT true,
    metadata        JSONB NOT NULL DEFAULT '{}',
    compliance_data JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, code)
);

CREATE TABLE customer (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    code            VARCHAR(100) NOT NULL,
    name            VARCHAR(500) NOT NULL,
    country_code    CHAR(2),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    metadata        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, code)
);
```

### Demand Forecasting

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
    model_details   JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_forecast_lookup ON demand_forecast(tenant_id, product_id, location_id, forecast_date);

CREATE TABLE demand_history (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    product_id      UUID NOT NULL REFERENCES product(id),
    location_id     UUID NOT NULL REFERENCES location(id),
    demand_date     DATE NOT NULL,
    quantity        NUMERIC(15, 2) NOT NULL,
    revenue         NUMERIC(15, 2),
    source          VARCHAR(50),
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
    logistics_data  JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, po_number)
);

CREATE TABLE purchase_order_line (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    purchase_order_id UUID NOT NULL REFERENCES purchase_order(id),
    line_number     INTEGER NOT NULL,
    product_id      UUID NOT NULL REFERENCES product(id),
    quantity        NUMERIC(15, 2) NOT NULL,
    unit_cost       NUMERIC(15, 4) NOT NULL,
    received_qty    NUMERIC(15, 2) DEFAULT 0,
    UNIQUE(tenant_id, purchase_order_id, line_number)
);
```

### KPIs & Audit

```sql
CREATE TABLE kpi_snapshot (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    period_start    DATE NOT NULL,
    period_end      DATE NOT NULL,
    scope_type      VARCHAR(20) NOT NULL DEFAULT 'company',
    scope_id        UUID,
    metrics         JSONB NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

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
```

---

## Graph Query Examples

### 1. Find all Tier 1 and Tier 2 suppliers for a product

```sql
-- "What is the full supplier network for product X?"
WITH RECURSIVE supplier_network AS (
    -- Base: Tier 1 suppliers
    SELECT 
        gn_supplier.entity_id AS supplier_id,
        gn_supplier.label AS supplier_name,
        ge.properties AS relationship,
        1 AS tier,
        ARRAY[gn_supplier.entity_id] AS path
    FROM graph_node gn_product
    JOIN graph_edge ge ON ge.target_node_id = gn_product.id AND ge.edge_type = 'SUPPLIES'
    JOIN graph_node gn_supplier ON gn_supplier.id = ge.source_node_id
    WHERE gn_product.tenant_id = :tenant_id
      AND gn_product.node_type = 'product'
      AND gn_product.entity_id = :product_id
      AND ge.is_active = true

    UNION ALL

    -- Recursive: sub-tier suppliers
    SELECT
        gn_sub.entity_id AS supplier_id,
        gn_sub.label AS supplier_name,
        ge_sub.properties AS relationship,
        sn.tier + 1 AS tier,
        sn.path || gn_sub.entity_id
    FROM supplier_network sn
    JOIN graph_node gn_parent ON gn_parent.entity_id = sn.supplier_id AND gn_parent.node_type = 'supplier'
    JOIN graph_edge ge_sub ON ge_sub.target_node_id = gn_parent.id AND ge_sub.edge_type = 'SUB_SUPPLIER_OF'
    JOIN graph_node gn_sub ON gn_sub.id = ge_sub.source_node_id
    WHERE ge_sub.is_active = true
      AND sn.tier < 5  -- limit depth
      AND NOT gn_sub.entity_id = ANY(sn.path)  -- prevent cycles
)
SELECT * FROM supplier_network ORDER BY tier, supplier_name;
```

### 2. Risk propagation: "What products are affected if Supplier X fails?"

```sql
-- Downstream impact analysis
WITH RECURSIVE impact_path AS (
    -- Start from the disrupted supplier
    SELECT 
        gn.id AS node_id,
        gn.node_type,
        gn.entity_id,
        gn.label,
        0 AS depth,
        ARRAY[gn.id] AS visited
    FROM graph_node gn
    WHERE gn.tenant_id = :tenant_id
      AND gn.node_type = 'supplier'
      AND gn.entity_id = :supplier_id

    UNION ALL

    -- Traverse downstream: supplier → product → location → customer
    SELECT
        gn_next.id,
        gn_next.node_type,
        gn_next.entity_id,
        gn_next.label,
        ip.depth + 1,
        ip.visited || gn_next.id
    FROM impact_path ip
    JOIN graph_edge ge ON ge.source_node_id = ip.node_id AND ge.is_active = true
    JOIN graph_node gn_next ON gn_next.id = ge.target_node_id
    WHERE ip.depth < 6
      AND NOT gn_next.id = ANY(ip.visited)
      AND ge.edge_type IN ('SUPPLIES', 'COMPONENT_OF', 'STORES_AT', 'SHIPS_TO', 'SOLD_TO')
)
SELECT node_type, entity_id, label, depth
FROM impact_path
WHERE node_type IN ('product', 'location', 'customer')
ORDER BY depth, node_type;
```

### 3. Find alternative suppliers for a product

```sql
-- "Supplier X is disrupted — who else can supply this product?"
SELECT 
    gn_alt.entity_id AS alt_supplier_id,
    gn_alt.label AS alt_supplier_name,
    ge_alt.properties->>'cost_premium_pct' AS cost_premium,
    ge_alt.properties->>'qualification_status' AS qualification,
    ge_supply.properties->>'lead_time_days' AS lead_time
FROM graph_node gn_current
-- Find ALTERNATIVE_TO edges from/to the current supplier
JOIN graph_edge ge_alt ON (
    (ge_alt.source_node_id = gn_current.id OR ge_alt.target_node_id = gn_current.id)
    AND ge_alt.edge_type = 'ALTERNATIVE_TO'
    AND ge_alt.is_active = true
)
JOIN graph_node gn_alt ON (
    gn_alt.id = CASE 
        WHEN ge_alt.source_node_id = gn_current.id THEN ge_alt.target_node_id
        ELSE ge_alt.source_node_id
    END
)
-- Check that alternative supplier also supplies the target product
JOIN graph_edge ge_supply ON (
    ge_supply.source_node_id = gn_alt.id
    AND ge_supply.edge_type = 'SUPPLIES'
    AND ge_supply.is_active = true
)
JOIN graph_node gn_product ON (
    gn_product.id = ge_supply.target_node_id
    AND gn_product.entity_id = :product_id
)
WHERE gn_current.tenant_id = :tenant_id
  AND gn_current.node_type = 'supplier'
  AND gn_current.entity_id = :disrupted_supplier_id;
```

### 4. Bill of Materials explosion

```sql
-- "What raw materials and components are needed to make product X?"
WITH RECURSIVE bom AS (
    SELECT 
        gn_component.entity_id AS component_id,
        gn_component.label AS component_name,
        (ge.properties->>'quantity_per')::NUMERIC AS qty_per,
        (ge.properties->>'bom_level')::INTEGER AS bom_level,
        1.0::NUMERIC AS cumulative_qty
    FROM graph_node gn_parent
    JOIN graph_edge ge ON ge.target_node_id = gn_parent.id AND ge.edge_type = 'COMPONENT_OF'
    JOIN graph_node gn_component ON gn_component.id = ge.source_node_id
    WHERE gn_parent.tenant_id = :tenant_id
      AND gn_parent.node_type = 'product'
      AND gn_parent.entity_id = :finished_product_id
      AND ge.is_active = true

    UNION ALL

    SELECT
        gn_sub.entity_id,
        gn_sub.label,
        (ge_sub.properties->>'quantity_per')::NUMERIC,
        (ge_sub.properties->>'bom_level')::INTEGER,
        bom.cumulative_qty * (ge_sub.properties->>'quantity_per')::NUMERIC
    FROM bom
    JOIN graph_node gn_mid ON gn_mid.entity_id = bom.component_id AND gn_mid.node_type = 'product'
    JOIN graph_edge ge_sub ON ge_sub.target_node_id = gn_mid.id AND ge_sub.edge_type = 'COMPONENT_OF'
    JOIN graph_node gn_sub ON gn_sub.id = ge_sub.source_node_id
    WHERE ge_sub.is_active = true
)
SELECT component_id, component_name, bom_level, cumulative_qty
FROM bom
ORDER BY bom_level, component_name;
```

### 5. Supplier concentration risk detection

```sql
-- "Which products depend on a single geographic region for >80% of supply?"
WITH product_supply AS (
    SELECT 
        gn_product.entity_id AS product_id,
        gn_product.label AS product_name,
        s.country_code,
        COUNT(*) AS supplier_count,
        SUM(CASE WHEN (ge.properties->>'is_preferred')::BOOLEAN THEN 1 ELSE 0 END) AS preferred_count
    FROM graph_node gn_product
    JOIN graph_edge ge ON ge.target_node_id = gn_product.id AND ge.edge_type = 'SUPPLIES' AND ge.is_active = true
    JOIN graph_node gn_supplier ON gn_supplier.id = ge.source_node_id
    JOIN supplier s ON s.id = gn_supplier.entity_id
    WHERE gn_product.tenant_id = :tenant_id
      AND gn_product.node_type = 'product'
    GROUP BY gn_product.entity_id, gn_product.label, s.country_code
),
product_totals AS (
    SELECT product_id, SUM(supplier_count) AS total_suppliers
    FROM product_supply
    GROUP BY product_id
)
SELECT 
    ps.product_id, ps.product_name, ps.country_code,
    ps.supplier_count, pt.total_suppliers,
    ROUND(ps.supplier_count::NUMERIC / pt.total_suppliers * 100, 1) AS pct_concentration
FROM product_supply ps
JOIN product_totals pt ON pt.product_id = ps.product_id
WHERE ps.supplier_count::NUMERIC / pt.total_suppliers > 0.8
ORDER BY pct_concentration DESC;
```

---

## Graph Synchronization

When master data changes in the relational tables, the graph must be updated:

```sql
-- Trigger to sync product changes to graph_node
CREATE OR REPLACE FUNCTION sync_product_to_graph() RETURNS TRIGGER AS $$
BEGIN
    IF TG_OP = 'INSERT' THEN
        INSERT INTO graph_node (tenant_id, node_type, entity_id, label, properties)
        VALUES (NEW.tenant_id, 'product', NEW.id, NEW.name, 
                jsonb_build_object('sku', NEW.sku, 'gtin', NEW.gtin, 'category', NEW.category));
    ELSIF TG_OP = 'UPDATE' THEN
        UPDATE graph_node 
        SET label = NEW.name, 
            properties = jsonb_build_object('sku', NEW.sku, 'gtin', NEW.gtin, 'category', NEW.category),
            updated_at = now()
        WHERE entity_id = NEW.id AND node_type = 'product';
    ELSIF TG_OP = 'DELETE' THEN
        -- Soft-delete: mark edges inactive rather than deleting
        UPDATE graph_edge SET is_active = false, updated_at = now()
        WHERE source_node_id IN (SELECT id FROM graph_node WHERE entity_id = OLD.id)
           OR target_node_id IN (SELECT id FROM graph_node WHERE entity_id = OLD.id);
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_product_graph_sync
    AFTER INSERT OR UPDATE OR DELETE ON product
    FOR EACH ROW EXECUTE FUNCTION sync_product_to_graph();
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Graph Layer | 2 | graph_node, graph_edge |
| Tenancy & Auth | 2 | tenant, app_user |
| Master Data | 4 | product, location, supplier, customer |
| Planning | 2 | demand_forecast, demand_history |
| Inventory | 1 | inventory_level |
| Procurement | 2 | purchase_order, purchase_order_line |
| Analytics & System | 2 | kpi_snapshot, audit_log |
| **Total** | **15** | Graph layer replaces ~6 junction/relationship tables from Model 1 |

---

## Key Design Decisions

1. **PostgreSQL-native graph, not Neo4j** — running a separate graph database adds operational complexity (two backups, two monitoring systems, two connection pools, cross-database consistency). PostgreSQL's recursive CTEs, JSONB properties on edges, and GIN indexes provide sufficient graph query performance for the expected data volumes (thousands of suppliers, tens of thousands of products). If graph traversal becomes a bottleneck at massive scale, the `graph_node` / `graph_edge` tables can be replicated to Neo4j without changing the relational layer.

2. **Generic node/edge tables vs. typed relationship tables** — a single `graph_edge` table with an `edge_type` column is more flexible than separate `supplier_product`, `location_link`, `bom_component` tables. New relationship types (e.g., `CERTIFIED_BY`, `INSURED_BY`) can be added without DDL changes. The trade-off is that referential integrity to specific entity types must be enforced at the application layer.

3. **Temporal edges with `valid_from` / `valid_to`** — supply chain relationships change over time (suppliers are onboarded, contracts expire, transport routes shift). The `valid_from` / `valid_to` columns enable point-in-time graph queries: "who were our suppliers for product X in Q1 2026?" Active edges are filtered with `WHERE is_active = true` and indexed accordingly.

4. **Weight column for path-finding** — the `weight` column on `graph_edge` enables weighted shortest-path algorithms. For transport links, weight could be transit time; for supplier relationships, weight could be inverse reliability (1 / OTIF rate). This powers queries like "what is the fastest alternative supply path?"

5. **Edge properties as JSONB** — each edge type carries different properties (lead time for SUPPLIES, quantity-per for COMPONENT_OF, transit days for SHIPS_TO). JSONB accommodates this variability without creating separate columns that would be NULL for most edge types.

6. **Relational tables remain the system of record** — the graph layer is a derived, synchronized view optimized for relationship queries. Product details, inventory levels, forecasts, and purchase orders are read from and written to relational tables. The graph is updated via triggers or application-level sync. This means standard CRUD operations don't pay the graph overhead.

7. **BOM as graph edges, not a separate table** — bill-of-materials relationships (component → parent product) are modeled as `COMPONENT_OF` edges rather than a dedicated BOM table. This enables unified graph traversal: "trace from raw material supplier through components to finished product to customer" in a single recursive query.

8. **Risk propagation via graph traversal** — when a supplier risk score changes (e.g., geopolitical score drops due to sanctions), a graph traversal from the supplier node identifies all downstream products, locations, and customers affected. This is the core differentiation of the graph model: risk analysis that would require multiple JOINs and application logic in a relational model becomes a single recursive query.

9. **Sub-supplier edges for CSDDD compliance** — the `SUB_SUPPLIER_OF` edge type captures Tier 2, Tier 3, etc. supplier relationships, which are required for EU CSDDD/LkSG due diligence. The normalized relational model (Model 1) can only represent direct supplier relationships naturally; multi-tier tracing requires awkward self-referential JOINs.

10. **Alternative supplier discovery** — `ALTERNATIVE_TO` edges between suppliers enable rapid alternative sourcing during disruptions. The graph query "find all approved alternative suppliers for product X who are not in the same geographic region as the disrupted supplier" composes naturally from edge traversals and node properties.
