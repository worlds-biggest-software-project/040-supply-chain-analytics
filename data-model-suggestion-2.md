# Data Model Suggestion 2: Event-Sourced / Audit-First (CQRS)

> Project: Supply Chain Analytics · Created: 2026-05-11

## Philosophy

This model treats every state change in the supply chain as an immutable event appended to a time-ordered event store. Rather than mutating rows (e.g., updating `inventory_level.on_hand_qty` from 500 to 480), the system records an event `InventoryDepleted { product_id, location_id, quantity: 20, reason: "sales_order_123" }`. The current state of any entity is derived by replaying its event stream from the beginning — or more practically, from the latest snapshot.

This is the CQRS (Command Query Responsibility Segregation) pattern: write operations append events to the event store (the "command side"), while read operations query pre-computed materialized views (the "query side"). The event store is the single source of truth; materialized views are disposable projections that can be rebuilt at any time by replaying events.

This approach is used by financial trading systems, healthcare audit systems, and increasingly by supply chain platforms like o9 Solutions (whose graph-based AI platform stores planning decisions as an event stream for full auditability). It is the natural fit when regulatory requirements (EU CSDDD, LkSG, ISO 9001) demand complete audit trails, when temporal queries ("what was the forecast on March 15?") are frequent, and when the platform needs to support AI-driven root cause analysis on the full history of supply chain decisions.

**Best for:** Platforms where full audit trail, temporal replay, regulatory compliance (CSDDD/LkSG/ISO 9001), and AI-powered analytics on decision history are primary requirements.

**Trade-offs:**
- (+) Complete, immutable audit trail by design — no separate audit table needed
- (+) Temporal queries are trivial: replay events up to any point in time
- (+) Supports "what if" analysis by branching event streams
- (+) AI/ML models can train on the full event history (every forecast revision, every override, every disruption response)
- (+) Schema evolution is natural: add new event types without altering existing tables
- (-) Higher storage requirements: events accumulate forever (mitigated by snapshots)
- (-) Read queries require materialized views; more infrastructure complexity
- (-) Developers unfamiliar with event sourcing face a learning curve
- (-) Eventual consistency between event store and read models requires careful handling
- (-) Debugging requires understanding event replay, not just querying current state

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| GS1 EPCIS 2.0 | Event structure directly mirrors EPCIS dimensions: What (entity), When (event_time), Where (location), Why (business_step), How (sensor/context data) |
| SCOR Digital Standard | SCOR process steps (Plan, Source, Make, Deliver, Return) map to event stream names |
| ISO 31000 | Risk assessment events capture risk identification, analysis, evaluation, treatment — the four ISO 31000 process steps |
| ISO 22301 / ISO/TS 22318 | Supply chain continuity events follow the disruption lifecycle: detection → assessment → response → recovery |
| CSDDD / LkSG | Immutable event store provides legally defensible audit evidence for supply chain due diligence |
| AsyncAPI 3.x | Event stream definitions documented as AsyncAPI specifications for integration partners |
| RFC 7807 | Error events use RFC 7807 Problem Details structure |

---

## Event Store (Core)

### Event Table

```sql
-- The single source of truth: an append-only event log
CREATE TABLE event_store (
    event_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    stream_name     VARCHAR(255) NOT NULL,      -- e.g. "product-{uuid}", "forecast-{uuid}", "po-{uuid}"
    stream_id       UUID NOT NULL,              -- the aggregate/entity this event belongs to
    event_type      VARCHAR(255) NOT NULL,       -- e.g. "ProductCreated", "ForecastGenerated", "InventoryAdjusted"
    event_version   INTEGER NOT NULL,            -- sequential per stream (optimistic concurrency)
    event_data      JSONB NOT NULL,              -- the event payload
    metadata        JSONB NOT NULL DEFAULT '{}', -- correlation_id, causation_id, user_id, ip_address
    event_time      TIMESTAMPTZ NOT NULL DEFAULT now(),  -- when the event occurred in the real world
    recorded_at     TIMESTAMPTZ NOT NULL DEFAULT now(),  -- when the event was recorded (bi-temporal)
    
    -- Optimistic concurrency: no two events with same version in a stream
    UNIQUE(tenant_id, stream_name, stream_id, event_version)
);

-- Primary query pattern: read all events for a stream in order
CREATE INDEX idx_event_stream ON event_store(tenant_id, stream_name, stream_id, event_version);

-- Time-range queries: "all events between date X and Y"
CREATE INDEX idx_event_time ON event_store(tenant_id, event_time);

-- Event type queries: "all ForecastOverrideApplied events"
CREATE INDEX idx_event_type ON event_store(tenant_id, event_type, event_time);

-- Partition by month for performance at scale
-- CREATE TABLE event_store_2026_05 PARTITION OF event_store
--     FOR VALUES FROM ('2026-05-01') TO ('2026-06-01');
```

### Snapshot Table

```sql
-- Periodic snapshots to avoid replaying entire event history
CREATE TABLE event_snapshot (
    snapshot_id     UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    stream_name     VARCHAR(255) NOT NULL,
    stream_id       UUID NOT NULL,
    snapshot_version INTEGER NOT NULL,           -- the event_version this snapshot is current to
    state           JSONB NOT NULL,              -- the full aggregate state at this version
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, stream_name, stream_id, snapshot_version)
);

CREATE INDEX idx_snapshot_stream ON event_snapshot(tenant_id, stream_name, stream_id, snapshot_version DESC);
```

---

## Event Type Catalog

The event store is schema-flexible: each `event_type` has a defined JSON structure, but new event types can be added without DDL changes. Here is the catalog of core event types:

### Product Events

```jsonc
// ProductCreated
{
    "product_id": "uuid",
    "sku": "WIDGET-001",
    "gtin": "00012345678905",
    "name": "Temperature Sensor Module",
    "category": "Electronics/Sensors",
    "unit_of_measure": "EA",
    "unit_cost": 12.50,
    "currency_code": "USD"
}

// ProductUpdated
{
    "product_id": "uuid",
    "changes": {
        "unit_cost": {"old": 12.50, "new": 13.75},
        "lead_time_days": {"old": 14, "new": 21}
    }
}

// ProductDeactivated
{
    "product_id": "uuid",
    "reason": "end_of_life"
}
```

### Forecast Events

```jsonc
// ForecastGenerated
{
    "generation_id": "uuid",
    "model_type": "temporal_fusion_transformer",
    "model_version": "2.3.1",
    "parameters": {"learning_rate": 0.001, "epochs": 100},
    "product_count": 1250,
    "location_count": 8,
    "horizon_days": 90,
    "overall_mape": 0.0823,
    "overall_wmape": 0.0654
}

// ForecastPointCreated (one per product/location/date in a generation)
{
    "generation_id": "uuid",
    "product_id": "uuid",
    "location_id": "uuid",
    "forecast_date": "2026-07-15",
    "bucket_type": "weekly",
    "quantity": 450.0,
    "confidence_lower": 380.0,
    "confidence_upper": 530.0
}

// ForecastOverrideApplied
{
    "forecast_point_id": "uuid",
    "generation_id": "uuid",
    "product_id": "uuid",
    "location_id": "uuid",
    "forecast_date": "2026-07-15",
    "original_qty": 450.0,
    "override_qty": 520.0,
    "reason": "Upcoming promotion in Q3 marketing calendar",
    "override_type": "promotion"
}

// ForecastOverrideApproved
{
    "override_event_id": "uuid",
    "approved_by": "uuid"
}
```

### Inventory Events

```jsonc
// InventoryReceived
{
    "product_id": "uuid",
    "location_id": "uuid",
    "quantity": 500,
    "batch_number": "LOT-2026-0511",
    "po_id": "uuid",
    "po_line_id": "uuid"
}

// InventoryDepleted
{
    "product_id": "uuid",
    "location_id": "uuid",
    "quantity": 20,
    "reason": "sales_fulfillment",
    "reference_id": "order-uuid"
}

// InventoryAdjusted
{
    "product_id": "uuid",
    "location_id": "uuid",
    "adjustment_qty": -5,
    "reason": "cycle_count_variance",
    "counted_by": "uuid"
}

// StockoutDetected
{
    "product_id": "uuid",
    "location_id": "uuid",
    "available_qty": 0,
    "demand_qty": 150,
    "estimated_stockout_duration_days": 7
}
```

### Purchase Order Events

```jsonc
// PurchaseOrderCreated
{
    "po_id": "uuid",
    "po_number": "PO-2026-0511-001",
    "supplier_id": "uuid",
    "ship_to_location_id": "uuid",
    "lines": [
        {"product_id": "uuid", "quantity": 1000, "unit_cost": 12.50}
    ],
    "expected_delivery_date": "2026-06-15",
    "total_amount": 12500.00,
    "currency_code": "USD"
}

// PurchaseOrderConfirmed (maps to EDI X12 855)
{
    "po_id": "uuid",
    "confirmed_delivery_date": "2026-06-18",
    "edi_reference": "855-20260512-001"
}

// ShipmentNotified (maps to EDI X12 856 ASN)
{
    "po_id": "uuid",
    "sscc": "00340123456789012345",
    "shipped_date": "2026-06-10",
    "carrier": "Maersk",
    "tracking_number": "MAEU1234567",
    "edi_reference": "856-20260610-001"
}

// PurchaseOrderReceived
{
    "po_id": "uuid",
    "received_lines": [
        {"product_id": "uuid", "ordered_qty": 1000, "received_qty": 985, "rejected_qty": 15}
    ],
    "actual_delivery_date": "2026-06-20"
}
```

### Supplier Risk Events

```jsonc
// SupplierRiskAssessed
{
    "supplier_id": "uuid",
    "overall_score": 72.5,
    "financial_score": 85.0,
    "operational_score": 70.0,
    "geopolitical_score": 55.0,
    "esg_score": 80.0,
    "concentration_score": 60.0,
    "methodology": "llm_enhanced",
    "signals": [
        {"source": "reuters", "headline": "Port strike threatens Q3 deliveries", "sentiment": -0.7},
        {"source": "esg_filing", "document": "2025 sustainability report", "rating": "B+"}
    ]
}

// SupplierDisruptionDetected
{
    "supplier_id": "uuid",
    "disruption_type": "logistics_delay",
    "severity": "high",
    "description": "Port congestion at Shanghai causing 14-day delays",
    "affected_po_ids": ["uuid1", "uuid2"],
    "detected_source": "news_feed_llm"
}

// DisruptionResponseRecommended
{
    "disruption_event_id": "uuid",
    "recommendations": [
        {"action": "expedite", "po_id": "uuid", "cost_impact": 2500.00},
        {"action": "re_source", "product_id": "uuid", "alt_supplier_id": "uuid", "lead_time_days": 7}
    ],
    "agent_model": "gpt-4o",
    "confidence": 0.85
}
```

---

## Materialized Read Models (Query Side)

These tables are projections built by processing the event stream. They can be rebuilt from scratch by replaying all events.

### Current Inventory View

```sql
CREATE TABLE view_inventory_current (
    tenant_id       UUID NOT NULL,
    product_id      UUID NOT NULL,
    location_id     UUID NOT NULL,
    on_hand_qty     NUMERIC(15, 2) NOT NULL DEFAULT 0,
    allocated_qty   NUMERIC(15, 2) NOT NULL DEFAULT 0,
    in_transit_qty  NUMERIC(15, 2) NOT NULL DEFAULT 0,
    last_event_id   UUID NOT NULL,              -- last event that updated this row
    last_event_time TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (tenant_id, product_id, location_id)
);
```

### Current Forecast View

```sql
CREATE TABLE view_forecast_current (
    tenant_id       UUID NOT NULL,
    product_id      UUID NOT NULL,
    location_id     UUID NOT NULL,
    forecast_date   DATE NOT NULL,
    quantity         NUMERIC(15, 2) NOT NULL,
    confidence_lower NUMERIC(15, 2),
    confidence_upper NUMERIC(15, 2),
    is_overridden   BOOLEAN NOT NULL DEFAULT false,
    override_qty    NUMERIC(15, 2),
    generation_id   UUID NOT NULL,
    model_type      VARCHAR(100),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (tenant_id, product_id, location_id, forecast_date)
);
```

### Current Purchase Order View

```sql
CREATE TABLE view_purchase_order_current (
    tenant_id       UUID NOT NULL,
    po_id           UUID NOT NULL,
    po_number       VARCHAR(100) NOT NULL,
    supplier_id     UUID NOT NULL,
    ship_to_location_id UUID NOT NULL,
    status          VARCHAR(30) NOT NULL,
    order_date      DATE NOT NULL,
    expected_delivery_date DATE,
    actual_delivery_date DATE,
    total_amount    NUMERIC(15, 2),
    currency_code   CHAR(3),
    line_count      INTEGER,
    last_event_id   UUID NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (tenant_id, po_id)
);

CREATE INDEX idx_view_po_supplier ON view_purchase_order_current(tenant_id, supplier_id);
CREATE INDEX idx_view_po_status ON view_purchase_order_current(tenant_id, status);
```

### Supplier Scorecard View

```sql
CREATE TABLE view_supplier_scorecard (
    tenant_id       UUID NOT NULL,
    supplier_id     UUID NOT NULL,
    current_risk_score NUMERIC(5, 2),
    avg_lead_time_days NUMERIC(8, 2),
    otif_rate_30d   NUMERIC(5, 4),
    otif_rate_90d   NUMERIC(5, 4),
    active_po_count INTEGER,
    active_disruption_count INTEGER,
    last_risk_assessment_date DATE,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (tenant_id, supplier_id)
);
```

### SCOR KPI View

```sql
CREATE TABLE view_kpi_scor (
    tenant_id       UUID NOT NULL,
    period_start    DATE NOT NULL,
    period_end      DATE NOT NULL,
    location_id     UUID,
    perfect_order_fulfillment  NUMERIC(5, 4),
    order_fulfillment_cycle_time NUMERIC(8, 2),
    fill_rate                  NUMERIC(5, 4),
    inventory_turns            NUMERIC(8, 2),
    inventory_days_of_supply   NUMERIC(8, 2),
    mape                       NUMERIC(8, 4),
    wmape                      NUMERIC(8, 4),
    stockout_rate              NUMERIC(5, 4),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (tenant_id, period_start, period_end, COALESCE(location_id, '00000000-0000-0000-0000-000000000000'::UUID))
);
```

---

## Reference Data Tables

Reference data is mutable (not event-sourced) because it changes infrequently and doesn't need temporal replay.

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
    UNIQUE(tenant_id, email)
);

CREATE TABLE product_ref (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    sku             VARCHAR(100) NOT NULL,
    gtin            VARCHAR(14),
    name            VARCHAR(500) NOT NULL,
    category        VARCHAR(500),
    unit_of_measure VARCHAR(20) NOT NULL DEFAULT 'EA',
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, sku)
);

CREATE TABLE location_ref (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    code            VARCHAR(100) NOT NULL,
    gln             VARCHAR(13),
    name            VARCHAR(500) NOT NULL,
    location_type   VARCHAR(50) NOT NULL,
    country_code    CHAR(2) NOT NULL,
    timezone        VARCHAR(50) DEFAULT 'UTC',
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, code)
);

CREATE TABLE supplier_ref (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    code            VARCHAR(100) NOT NULL,
    name            VARCHAR(500) NOT NULL,
    country_code    CHAR(2),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, code)
);
```

---

## Temporal Query Examples

### "What was the inventory level for product X at location Y on March 15?"

```sql
-- Replay inventory events up to a specific point in time
SELECT 
    SUM(CASE 
        WHEN event_type = 'InventoryReceived' THEN (event_data->>'quantity')::NUMERIC
        WHEN event_type = 'InventoryDepleted' THEN -(event_data->>'quantity')::NUMERIC
        WHEN event_type = 'InventoryAdjusted' THEN (event_data->>'adjustment_qty')::NUMERIC
        ELSE 0
    END) AS on_hand_qty
FROM event_store
WHERE tenant_id = :tenant_id
  AND stream_name = 'inventory'
  AND event_data->>'product_id' = :product_id
  AND event_data->>'location_id' = :location_id
  AND event_time <= '2026-03-15T23:59:59Z'
  AND event_type IN ('InventoryReceived', 'InventoryDepleted', 'InventoryAdjusted');
```

### "What forecasts were active when we placed PO-2026-0301?"

```sql
-- Find the forecast generation that was active at the time of a PO
WITH po_event AS (
    SELECT event_time 
    FROM event_store
    WHERE tenant_id = :tenant_id
      AND event_type = 'PurchaseOrderCreated'
      AND event_data->>'po_number' = 'PO-2026-0301'
    LIMIT 1
)
SELECT e.event_data
FROM event_store e, po_event p
WHERE e.tenant_id = :tenant_id
  AND e.event_type = 'ForecastGenerated'
  AND e.event_time <= p.event_time
ORDER BY e.event_time DESC
LIMIT 1;
```

### "Show the complete decision trail for a disruption response"

```sql
-- Full event chain: disruption detected → recommendations → approvals
SELECT event_type, event_data, event_time, metadata->>'user_id' AS actor
FROM event_store
WHERE tenant_id = :tenant_id
  AND (
    event_data->>'disruption_event_id' = :disruption_id
    OR event_id = :disruption_id::UUID
  )
ORDER BY event_time ASC;
```

---

## Event Processing Architecture

```
┌─────────────┐     ┌──────────────┐     ┌─────────────────────┐
│  REST API   │────▶│ Command      │────▶│ Event Store          │
│  (writes)   │     │ Handler      │     │ (append-only table)  │
└─────────────┘     └──────────────┘     └─────────┬───────────┘
                                                    │
                                          ┌─────────▼───────────┐
                                          │ Event Projector     │
                                          │ (async worker)      │
                                          └─────────┬───────────┘
                                                    │
                    ┌───────────────────────────────┼───────────────────┐
                    ▼                               ▼                   ▼
          ┌─────────────────┐           ┌───────────────────┐  ┌──────────────┐
          │ view_inventory  │           │ view_forecast     │  │ view_kpi     │
          │ _current        │           │ _current          │  │ _scor        │
          └─────────────────┘           └───────────────────┘  └──────────────┘
                    ▲                               ▲                   ▲
                    │                               │                   │
          ┌─────────────────────────────────────────────────────────────┐
          │                      REST API (reads)                       │
          └─────────────────────────────────────────────────────────────┘
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Store | 2 | event_store (partitioned), event_snapshot |
| Materialized Views | 5 | inventory, forecast, PO, supplier scorecard, SCOR KPIs |
| Reference Data | 5 | tenant, app_user, product_ref, location_ref, supplier_ref |
| **Total** | **12** | Plus event type catalog (no DDL changes needed) |

---

## Key Design Decisions

1. **Single event table vs. multiple streams** — all events live in one `event_store` table, differentiated by `stream_name` and `event_type`. This simplifies infrastructure (one table to back up, partition, and index) while the `stream_name` + `stream_id` compound index provides efficient per-aggregate queries.

2. **Bi-temporal timestamps** — `event_time` (when the event occurred in the real world) and `recorded_at` (when it was persisted) are both stored. This supports backdated corrections: "we received the goods on June 15 but didn't record it until June 18."

3. **JSONB event payloads** — event data is stored as JSONB rather than separate columns, enabling schema evolution without DDL migrations. New event types are simply new JSON shapes. GIN indexes on `event_data` support efficient containment queries.

4. **Optimistic concurrency via event_version** — the `UNIQUE(tenant_id, stream_name, stream_id, event_version)` constraint prevents conflicting writes. If two processes try to append event version 5 to the same stream, one will fail with a unique violation — the application retries with the current version.

5. **Snapshots for performance** — rather than replaying thousands of events to reconstruct an aggregate, periodic snapshots store the full state at a given version. Reconstruction then replays only events after the snapshot.

6. **Mutable reference data** — products, locations, and suppliers are stored in traditional mutable tables because they change infrequently and temporal replay of reference data adds complexity without proportional value. Product changes are still captured as `ProductUpdated` events for auditing.

7. **Materialized views are disposable** — every `view_*` table can be dropped and rebuilt by replaying the event store. This means schema changes to read models are zero-risk: deploy new projector code, rebuild the view, swap traffic.

8. **EPCIS alignment** — the event structure intentionally mirrors EPCIS 2.0's five dimensions (What/When/Where/Why/How), making it straightforward to export events as EPCIS-compliant JSON for trading partner integration and FDA FSMA traceability compliance.

9. **Partitioning by time** — the event store should be range-partitioned by `event_time` (monthly or quarterly) for performance. Old partitions can be moved to cold storage (S3/GCS) without affecting active queries.

10. **Event-driven AI training** — the complete event history is a natural training dataset for ML models. Forecast accuracy can be evaluated retroactively by comparing `ForecastPointCreated` events against later `InventoryDepleted` events, without needing a separate analytics warehouse.
