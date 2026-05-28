# Supply Chain Analytics -- Development Plan

> Project: Supply Chain Analytics (Candidate #40)
> Created: 2026-05-25
> Based on: research.md, features.md, standards.md, data-model-suggestion-1..4, README.md

---

## Technology Decisions & Rationale

### Data Model: Hybrid Relational + JSONB (Model 3) with Graph Extension (Model 4)

**Decision:** Adopt Data Model Suggestion 3 (Hybrid Relational + JSONB) as the primary schema, and introduce the Graph Layer (Model 4's `graph_node`/`graph_edge` tables) in Phase 6 when supplier network analysis and risk propagation features are built.

**Rationale:**
- Model 3's ~15 core tables provide the fastest path to MVP while retaining relational integrity for the core supply chain domain (products, locations, forecasts, inventory, purchase orders).
- JSONB columns on master data entities absorb industry-specific variability (pharma batch tracking, food allergens, electronics tariff codes) without DDL migrations -- critical for a platform targeting diverse mid-market verticals.
- Model 1's full 3NF normalization (23+ tables) adds schema complexity that slows iteration in early phases without proportional benefit until scale demands it.
- Model 2's event sourcing adds infrastructure overhead (event projectors, snapshot management, eventual consistency) that is premature before product-market fit is established.
- Model 4's graph layer is valuable but only becomes essential when multi-tier supplier network analysis and risk propagation features are built (Phase 6+). Adding `graph_node` and `graph_edge` tables at that point is additive, not disruptive.

### Backend: Python (FastAPI) + PostgreSQL

**Decision:** Python 3.12+ with FastAPI for the API layer; PostgreSQL 16+ as the sole database.

**Rationale:**
- Python is the dominant language for ML/data science. The core differentiator of this platform -- ML-based demand forecasting with LSTM/TFT models -- requires PyTorch, scikit-learn, LightGBM, and SHAP. Using Python for both the API and ML pipeline eliminates a language boundary.
- FastAPI provides async I/O, automatic OpenAPI 3.1 spec generation (standards.md requirement), and Pydantic-based validation.
- PostgreSQL is the only database needed: relational tables for CRUD, JSONB for flexible metadata, GIN indexes for containment queries, row-level security for multi-tenancy, and native recursive CTEs for the graph layer in Phase 6. Avoiding a second database (Neo4j, Redis, etc.) reduces operational complexity.
- Alembic for schema migrations with a strict "one migration per PR" policy.

### ML/Forecasting: LightGBM (MVP) then Temporal Fusion Transformer (Phase 4)

**Decision:** Start with LightGBM gradient boosting for demand forecasting in Phase 2, then add PyTorch-based Temporal Fusion Transformer (TFT) in Phase 4.

**Rationale:**
- LightGBM trains in seconds on mid-market data volumes (thousands of SKUs, not millions), requires no GPU, and consistently outperforms ARIMA/exponential smoothing on tabular demand data. It is the pragmatic starting model.
- TFT is the state-of-the-art for multi-horizon probabilistic forecasting with attention-based explainability, but requires more training data, GPU resources, and engineering effort. Deferring to Phase 4 allows the platform to ship with competitive forecasting in MVP.
- SHAP explainability is integrated from Phase 2 onward -- a key differentiator over Black-box enterprise tools (Blue Yonder, Kinaxis).

### Frontend: React 18 + TypeScript + Recharts

**Decision:** React 18 with TypeScript for the web UI; Recharts for charting; Tailwind CSS for styling.

**Rationale:**
- React + TypeScript is the dominant frontend stack for data-heavy SaaS applications. The planning dashboard, forecast charts, inventory grids, and scenario modeling UI all require performant, reactive data binding.
- Recharts is a React-native charting library that renders forecast confidence intervals, time-series comparisons, and KPI sparklines without the weight of D3.js.
- Tailwind CSS enables rapid UI iteration without maintaining a separate design system in early phases.

### Authentication: OAuth 2.0 + JWT + Optional OIDC SSO

**Decision:** JWT-based authentication with OAuth 2.0 authorization; OpenID Connect SSO support added in Phase 5.

**Rationale:**
- JWT (RFC 7519) is the industry standard for API authentication in supply chain SaaS (standards.md: all commercial platforms use JWT/OAuth 2.0).
- OIDC SSO is deferred to Phase 5 because mid-market MVPs rarely require enterprise identity federation at launch, but it is a hard requirement for enterprise sales.

### API Design: REST/JSON + OpenAPI 3.1 + AsyncAPI 3.x

**Decision:** REST/JSON API documented with OpenAPI 3.1; event-driven endpoints (inventory alerts, disruption notifications) documented with AsyncAPI 3.x when WebSocket/SSE support is added in Phase 5.

**Rationale:**
- OpenAPI 3.1 is required for developer tooling compatibility (standards.md). Auto-generated from FastAPI route decorators.
- AsyncAPI documentation is deferred until real-time features are built, but the AsyncAPI spec structure informs event payload design from Phase 1.
- Error responses follow RFC 7807 (Problem Details for HTTP APIs) from day one.

### Licensing: MIT

**Decision:** MIT license for the open-source core.

**Rationale:**
- MIT is the most permissive widely-recognized license, maximizing adoption. frePPLe's switch from AGPL to MIT at v8.0 (features.md) validates this approach for supply chain OSS. Apache 2.0 was considered but MIT's simplicity is preferred for developer adoption.

---

## Project Structure

```
supply-chain-analytics/
  README.md
  LICENSE                          # MIT
  docker-compose.yml               # PostgreSQL + API + Worker + Frontend
  docker-compose.dev.yml           # Development overrides
  .env.example
  alembic.ini
  pyproject.toml                   # Python project config (uv/poetry)

  backend/
    app/
      __init__.py
      main.py                      # FastAPI application factory
      config.py                    # Settings (pydantic-settings)
      database.py                  # SQLAlchemy async engine + session
      dependencies.py              # FastAPI dependency injection

      models/                      # SQLAlchemy ORM models
        __init__.py
        tenant.py                  # tenant, app_user
        product.py                 # product
        location.py                # location
        supplier.py                # supplier
        customer.py                # customer
        demand.py                  # demand_forecast, demand_history
        inventory.py               # inventory_level, inventory_policy
        procurement.py             # purchase_order, purchase_order_line
        analytics.py               # supplier_risk_assessment, kpi_snapshot
        system.py                  # audit_log, data_import
        graph.py                   # graph_node, graph_edge (Phase 6)

      schemas/                     # Pydantic request/response schemas
        __init__.py
        product.py
        location.py
        supplier.py
        demand.py
        inventory.py
        procurement.py
        analytics.py

      api/                         # FastAPI routers
        __init__.py
        v1/
          __init__.py
          products.py
          locations.py
          suppliers.py
          forecasts.py
          inventory.py
          purchase_orders.py
          kpis.py
          imports.py
          scenarios.py             # Phase 3
          risk.py                  # Phase 5
          graph.py                 # Phase 6

      services/                    # Business logic layer
        __init__.py
        product_service.py
        forecast_service.py
        inventory_service.py
        procurement_service.py
        import_service.py
        kpi_service.py
        scenario_service.py        # Phase 3
        risk_service.py            # Phase 5
        graph_service.py           # Phase 6

      ml/                          # Machine learning pipeline
        __init__.py
        pipeline.py                # Orchestration: train, predict, evaluate
        feature_engineering.py     # Feature extraction from demand history
        models/
          __init__.py
          lightgbm_model.py        # Phase 2
          tft_model.py             # Phase 4 (Temporal Fusion Transformer)
          ensemble.py              # Phase 4 (model ensembling)
        explainability.py          # SHAP value computation
        evaluation.py              # MAPE, WMAPE, bias calculations
        external_signals.py        # Weather, social, macro data (Phase 4)

      workers/                     # Background task processing
        __init__.py
        forecast_worker.py         # Async forecast generation
        kpi_worker.py              # Periodic KPI snapshot computation
        import_worker.py           # CSV/API data import processing
        risk_worker.py             # Supplier risk scoring (Phase 5)
        disruption_worker.py       # Disruption detection agent (Phase 7)

      integrations/                # External system connectors
        __init__.py
        csv_importer.py
        erp_connector.py           # Phase 5
        edi_parser.py              # Phase 8

      middleware/
        __init__.py
        tenant.py                  # Tenant resolution from JWT/header
        audit.py                   # Audit log middleware

    migrations/                    # Alembic migrations
      versions/
      env.py

    tests/
      __init__.py
      conftest.py                  # Fixtures: test DB, tenant, user, sample data
      factories.py                 # Factory Boy factories for test data
      unit/
        test_forecast_service.py
        test_inventory_service.py
        test_kpi_service.py
        test_import_service.py
      integration/
        test_product_api.py
        test_forecast_api.py
        test_inventory_api.py
        test_import_api.py
      ml/
        test_lightgbm_model.py
        test_feature_engineering.py
        test_evaluation.py

  frontend/
    package.json
    tsconfig.json
    vite.config.ts
    src/
      App.tsx
      main.tsx
      api/                         # API client (generated from OpenAPI)
      components/
        layout/
        dashboard/
        forecasts/
        inventory/
        suppliers/
        scenarios/                 # Phase 3
        risk/                      # Phase 5
      hooks/
      pages/
      store/                       # Zustand state management
      types/
      utils/
    tests/
      components/
      pages/
```

---

## Phase Dependency Graph

```
Phase 1: Foundation
    |
    v
Phase 2: Demand Forecasting ------+
    |                              |
    v                              v
Phase 3: Scenario Modeling    Phase 4: Advanced ML
    |                              |
    +------+-----------+-----------+
           |           |
           v           v
      Phase 5: Supplier     Phase 6: Graph Layer &
      Performance &         Network Analysis
      Risk Scoring               |
           |                     |
           +----------+----------+
                      |
                      v
              Phase 7: Agentic AI &
              Disruption Response
                      |
                      v
              Phase 8: Enterprise
              Integration & Compliance
                      |
                      v
              Phase 9: S&OP Intelligence
                      |
                      v
              Phase 10: Multi-Enterprise
              & Sustainability
```

**Critical path:** Phase 1 -> Phase 2 -> Phase 4 -> Phase 7 (core AI differentiation)

**Parallel tracks after Phase 2:**
- Track A (Planning depth): Phase 3 (Scenarios) -> Phase 9 (S&OP)
- Track B (ML depth): Phase 4 (Advanced ML) -> Phase 7 (Agentic AI)
- Track C (Supplier intelligence): Phase 5 (Supplier Performance) -> Phase 6 (Graph) -> Phase 7 (Agentic AI)

---

## Phase 1: Foundation & Core Platform

**Duration:** 4-5 weeks
**Dependencies:** None (starting phase)
**Goal:** Multi-tenant API with master data CRUD, CSV import, basic authentication, and a working frontend shell.

### Task 1.1: Database Schema & Migrations

**What:** Create the PostgreSQL schema for tenancy, master data (product, location, supplier, customer), and system tables (audit_log, data_import). Apply row-level security.

**Design:**
```python
# backend/app/models/tenant.py
from sqlalchemy import Column, String, UUID, TIMESTAMP, text
from sqlalchemy.dialects.postgresql import JSONB
from app.database import Base

class Tenant(Base):
    __tablename__ = "tenant"
    id = Column(UUID, primary_key=True, server_default=text("gen_random_uuid()"))
    name = Column(String(255), nullable=False)
    slug = Column(String(100), nullable=False, unique=True)
    plan_tier = Column(String(50), nullable=False, server_default="free")
    industry = Column(String(100))
    settings = Column(JSONB, nullable=False, server_default=text("'{}'::jsonb"))
    created_at = Column(TIMESTAMP(timezone=True), server_default=text("now()"))
    updated_at = Column(TIMESTAMP(timezone=True), server_default=text("now()"))

# backend/app/models/product.py
class Product(Base):
    __tablename__ = "product"
    id = Column(UUID, primary_key=True, server_default=text("gen_random_uuid()"))
    tenant_id = Column(UUID, ForeignKey("tenant.id"), nullable=False)
    sku = Column(String(100), nullable=False)
    gtin = Column(String(14))
    name = Column(String(500), nullable=False)
    category = Column(String(500))
    unit_of_measure = Column(String(20), nullable=False, server_default="EA")
    unit_cost = Column(Numeric(15, 4))
    currency_code = Column(String(3), server_default="USD")
    lead_time_days = Column(Integer)
    is_active = Column(Boolean, nullable=False, server_default=text("true"))
    metadata = Column(JSONB, nullable=False, server_default=text("'{}'::jsonb"))
    created_at = Column(TIMESTAMP(timezone=True), server_default=text("now()"))
    updated_at = Column(TIMESTAMP(timezone=True), server_default=text("now()"))
    __table_args__ = (UniqueConstraint("tenant_id", "sku"),)

# Alembic migration applies RLS:
# ALTER TABLE product ENABLE ROW LEVEL SECURITY;
# CREATE POLICY tenant_isolation ON product
#     USING (tenant_id = current_setting('app.current_tenant_id')::UUID);
```

**Testing:**
- Migration runs cleanly on empty database (`alembic upgrade head`)
- Migration is reversible (`alembic downgrade -1`)
- RLS prevents cross-tenant data access: insert rows for tenant A, set session to tenant B, verify SELECT returns zero rows
- Unique constraints enforced: inserting duplicate `(tenant_id, sku)` raises IntegrityError
- JSONB metadata column accepts arbitrary JSON and is queryable with `@>` operator
- All `created_at` columns auto-populate with current timestamp

### Task 1.2: Authentication & Authorization

**What:** JWT-based authentication with role-based access control (admin, planner, viewer, api).

**Design:**
```python
# backend/app/api/v1/auth.py
from fastapi import APIRouter, Depends, HTTPException
from fastapi.security import OAuth2PasswordBearer
from jose import jwt, JWTError
from passlib.context import CryptContext

router = APIRouter(prefix="/auth", tags=["auth"])

@router.post("/login")
async def login(credentials: LoginRequest, db: AsyncSession = Depends(get_db)):
    """Authenticate user, return JWT with tenant_id and role claims."""
    user = await authenticate_user(db, credentials.email, credentials.password)
    if not user:
        raise HTTPException(status_code=401, detail="Invalid credentials")
    token = create_access_token(data={
        "sub": str(user.id),
        "tenant_id": str(user.tenant_id),
        "role": user.role,
    })
    return {"access_token": token, "token_type": "bearer"}

@router.post("/register")
async def register_tenant(request: RegisterRequest, db: AsyncSession = Depends(get_db)):
    """Create tenant + admin user in a single transaction."""
    ...

# backend/app/middleware/tenant.py
async def set_tenant_context(db: AsyncSession, tenant_id: str):
    """Set PostgreSQL session variable for RLS enforcement."""
    await db.execute(text(f"SET app.current_tenant_id = '{tenant_id}'"))
```

**Testing:**
- POST `/auth/register` creates tenant + admin user, returns valid JWT
- POST `/auth/login` with valid credentials returns JWT with correct `tenant_id` and `role` claims
- POST `/auth/login` with invalid password returns 401
- JWT expiry is enforced: expired token returns 401
- Role-based access: `viewer` role cannot POST/PUT/DELETE on product endpoints; `planner` can; `admin` can manage users
- Tenant isolation via JWT: user from tenant A cannot access tenant B resources even with valid JWT (RLS enforced)
- API key authentication (role=`api`) works for machine-to-machine integration

### Task 1.3: Master Data REST API

**What:** CRUD endpoints for products, locations, suppliers, customers with pagination, filtering, and JSONB metadata support.

**Design:**
```python
# backend/app/api/v1/products.py
from fastapi import APIRouter, Depends, Query
from app.schemas.product import ProductCreate, ProductUpdate, ProductResponse, ProductList

router = APIRouter(prefix="/products", tags=["products"])

@router.get("/", response_model=ProductList)
async def list_products(
    page: int = Query(1, ge=1),
    page_size: int = Query(50, ge=1, le=500),
    category: str | None = None,
    is_active: bool | None = None,
    search: str | None = None,
    db: AsyncSession = Depends(get_db),
    current_user: AppUser = Depends(get_current_user),
):
    """List products with pagination and filtering."""
    ...

@router.post("/", response_model=ProductResponse, status_code=201)
async def create_product(
    product: ProductCreate,
    db: AsyncSession = Depends(get_db),
    current_user: AppUser = Depends(require_role(["admin", "planner"])),
):
    """Create a new product. Metadata JSONB accepts any valid JSON."""
    ...

@router.get("/{product_id}", response_model=ProductResponse)
async def get_product(product_id: UUID, ...):
    """Get product by ID. Returns 404 if not found or belongs to different tenant."""
    ...

@router.put("/{product_id}", response_model=ProductResponse)
async def update_product(product_id: UUID, product: ProductUpdate, ...):
    """Update product. Writes audit_log entry with old/new diff."""
    ...

@router.delete("/{product_id}", status_code=204)
async def delete_product(product_id: UUID, ...):
    """Soft-delete (set is_active=false). Hard delete only for admin."""
    ...
```

**Testing:**
- CRUD operations: Create product, read back, update fields, soft-delete, verify `is_active=false`
- Pagination: Create 120 products, request page 3 with page_size 50, verify 20 results returned with correct `total_count`
- Filtering: Create products in categories "Electronics" and "Food", filter by `category=Electronics`, verify only matching products returned
- Search: Create product named "Temperature Sensor", search `?search=temp`, verify match
- JSONB metadata: Create product with `metadata={"hs_code": "8542.31", "rohs_compliant": true}`, verify stored and retrievable
- GTIN validation: 14-digit GTIN check-digit validation on create/update
- Audit trail: After update, verify `audit_log` contains entry with old/new values as JSONB diff
- Tenant isolation: Product created by tenant A is not returned in tenant B's GET /products
- OpenAPI schema: Verify `/docs` renders all endpoints with correct request/response schemas

### Task 1.4: CSV Data Import

**What:** Upload CSV files to bulk-import products, locations, suppliers, and demand history. Validate rows, report errors, and track import status.

**Design:**
```python
# backend/app/api/v1/imports.py
@router.post("/imports", response_model=DataImportResponse, status_code=202)
async def start_import(
    file: UploadFile,
    entity_type: str = Query(..., regex="^(product|location|supplier|demand_history)$"),
    db: AsyncSession = Depends(get_db),
    current_user: AppUser = Depends(require_role(["admin", "planner"])),
):
    """
    Accept CSV upload, create data_import record, enqueue background processing.
    Returns import_id for status polling.
    """
    import_record = DataImport(
        tenant_id=current_user.tenant_id,
        source_type="csv",
        entity_type=entity_type,
        status="pending",
        import_data={"file_name": file.filename, "column_mapping": {}},
    )
    db.add(import_record)
    await db.commit()
    # Enqueue to background worker
    await enqueue_import(import_record.id, file_contents)
    return import_record

@router.get("/imports/{import_id}", response_model=DataImportStatus)
async def get_import_status(import_id: UUID, ...):
    """Poll import status. Returns row_count, success_count, error_count, errors[]."""
    ...

# backend/app/workers/import_worker.py
async def process_csv_import(import_id: UUID):
    """
    Read CSV, validate each row against Pydantic schema,
    upsert valid rows, collect errors with row numbers.
    """
    for row_num, row in enumerate(csv_reader, start=1):
        try:
            validated = ProductCreate(**row)
            await upsert_product(db, validated)
            success_count += 1
        except ValidationError as e:
            errors.append({"row": row_num, "errors": e.errors()})
            error_count += 1
    ...
```

**Testing:**
- Upload valid 100-row product CSV, verify all 100 products created, import status shows `success_count=100, error_count=0`
- Upload CSV with 5 invalid rows (missing required fields), verify 95 products created, `error_count=5`, errors array contains row numbers and field-level error messages
- Upload demand_history CSV with 10,000 rows, verify import completes within 30 seconds
- Duplicate SKU handling: Upload CSV with duplicate SKU, verify upsert behavior (update existing, not fail)
- Column mapping: CSV with non-standard headers (e.g., "Item Code" instead of "sku") uses `column_mapping` from import_data JSONB
- Status polling: GET `/imports/{id}` returns current progress during processing
- Empty file returns 400 with descriptive error
- Non-CSV file (e.g., .xlsx) returns 400 with "unsupported file type" error

### Task 1.5: Frontend Shell & Dashboard Layout

**What:** React application with authentication flow, sidebar navigation, and a placeholder dashboard page.

**Design:**
```typescript
// frontend/src/App.tsx
import { BrowserRouter, Routes, Route, Navigate } from "react-router-dom";
import { AuthProvider } from "./hooks/useAuth";
import { ProtectedRoute } from "./components/layout/ProtectedRoute";
import { AppLayout } from "./components/layout/AppLayout";
import { LoginPage } from "./pages/LoginPage";
import { DashboardPage } from "./pages/DashboardPage";
import { ProductsPage } from "./pages/ProductsPage";
import { InventoryPage } from "./pages/InventoryPage";

export default function App() {
  return (
    <AuthProvider>
      <BrowserRouter>
        <Routes>
          <Route path="/login" element={<LoginPage />} />
          <Route element={<ProtectedRoute><AppLayout /></ProtectedRoute>}>
            <Route path="/" element={<Navigate to="/dashboard" />} />
            <Route path="/dashboard" element={<DashboardPage />} />
            <Route path="/products" element={<ProductsPage />} />
            <Route path="/inventory" element={<InventoryPage />} />
          </Route>
        </Routes>
      </BrowserRouter>
    </AuthProvider>
  );
}

// frontend/src/components/layout/AppLayout.tsx
// Sidebar navigation: Dashboard, Products, Locations, Suppliers, Inventory,
// Forecasts (Phase 2), Scenarios (Phase 3), Supplier Risk (Phase 5)
```

**Testing:**
- Login page renders, submits credentials, stores JWT in localStorage, redirects to dashboard
- Invalid credentials show error message without redirect
- Sidebar navigation renders all menu items; clicking navigates to correct route
- Unauthenticated access to `/dashboard` redirects to `/login`
- Products page renders data table with pagination controls
- Responsive layout: sidebar collapses on mobile viewport (< 768px)
- Logout clears JWT and redirects to login

### Definition of Done -- Phase 1
- [ ] PostgreSQL schema created and migrated with RLS on all tenant-scoped tables
- [ ] JWT authentication works for login, registration, and all API endpoints
- [ ] CRUD APIs for products, locations, suppliers, customers pass all tests
- [ ] CSV import processes files asynchronously with error reporting
- [ ] Frontend login flow, sidebar navigation, and product listing work end-to-end
- [ ] OpenAPI spec auto-generated and accessible at `/docs`
- [ ] All error responses follow RFC 7807 Problem Details format
- [ ] Audit log records all create/update/delete operations
- [ ] Docker Compose starts entire stack (PostgreSQL, API, frontend) with one command
- [ ] Test suite: >= 80% backend code coverage; all integration tests pass

---

## Phase 2: Demand Forecasting Engine

**Duration:** 4-5 weeks
**Dependencies:** Phase 1 (master data, import, auth)
**Goal:** ML-based demand forecasting with LightGBM, forecast accuracy metrics, planner override workflow, and forecast visualization.

### Task 2.1: Demand History Ingestion & Feature Engineering

**What:** Ingest demand history data (CSV or API), compute time-series features (lags, rolling means, seasonal indicators) for ML training.

**Design:**
```python
# backend/app/ml/feature_engineering.py
import pandas as pd
import numpy as np

def build_features(demand_df: pd.DataFrame) -> pd.DataFrame:
    """
    Input: DataFrame with columns [product_id, location_id, demand_date, quantity]
    Output: DataFrame with engineered features for ML model input.
    """
    df = demand_df.sort_values(["product_id", "location_id", "demand_date"])
    grouped = df.groupby(["product_id", "location_id"])

    # Lag features
    for lag in [7, 14, 28, 56, 91]:
        df[f"lag_{lag}d"] = grouped["quantity"].shift(lag)

    # Rolling statistics
    for window in [7, 14, 28]:
        df[f"rolling_mean_{window}d"] = grouped["quantity"].transform(
            lambda x: x.rolling(window, min_periods=1).mean()
        )
        df[f"rolling_std_{window}d"] = grouped["quantity"].transform(
            lambda x: x.rolling(window, min_periods=1).std()
        )

    # Calendar features
    df["day_of_week"] = pd.to_datetime(df["demand_date"]).dt.dayofweek
    df["month"] = pd.to_datetime(df["demand_date"]).dt.month
    df["week_of_year"] = pd.to_datetime(df["demand_date"]).dt.isocalendar().week.astype(int)
    df["is_month_end"] = pd.to_datetime(df["demand_date"]).dt.is_month_end.astype(int)

    # Trend features
    df["days_since_start"] = (
        pd.to_datetime(df["demand_date"]) - pd.to_datetime(df["demand_date"]).min()
    ).dt.days

    return df.dropna()  # Drop rows where lag features are NaN
```

**Testing:**
- Feature engineering on 365-day daily demand history for 1 product produces correct number of lag columns (5 lags) and rolling stat columns (6 rolling features)
- Lag values are mathematically correct: `lag_7d` for day 8 equals quantity on day 1
- Rolling mean for 7-day window on days 1-7 equals the mean of those 7 values
- Calendar features are correct: January 1 has `month=1`, `day_of_week=correct_value`
- NaN rows dropped: first 91 rows (max lag) are excluded from output
- Multi-product input: features computed independently per `(product_id, location_id)` group -- no data leakage between products
- Empty demand history raises descriptive ValueError

### Task 2.2: LightGBM Demand Forecasting Model

**What:** Train LightGBM model on demand history features, generate point forecasts + confidence intervals, compute accuracy metrics.

**Design:**
```python
# backend/app/ml/models/lightgbm_model.py
import lightgbm as lgb
from sklearn.model_selection import TimeSeriesSplit

class LightGBMForecaster:
    def __init__(self, params: dict | None = None):
        self.params = params or {
            "objective": "regression",
            "metric": "mae",
            "num_leaves": 31,
            "learning_rate": 0.05,
            "feature_fraction": 0.8,
            "bagging_fraction": 0.8,
            "bagging_freq": 5,
            "verbose": -1,
        }
        self.model = None

    def train(self, X_train: pd.DataFrame, y_train: pd.Series,
              X_val: pd.DataFrame, y_val: pd.Series) -> dict:
        """Train model, return training metrics."""
        train_set = lgb.Dataset(X_train, label=y_train)
        val_set = lgb.Dataset(X_val, label=y_val, reference=train_set)
        self.model = lgb.train(
            self.params, train_set,
            valid_sets=[val_set],
            num_boost_round=1000,
            callbacks=[lgb.early_stopping(50), lgb.log_evaluation(100)],
        )
        return {"best_iteration": self.model.best_iteration}

    def predict(self, X: pd.DataFrame) -> dict:
        """Generate point forecast + confidence intervals (P10, P90)."""
        point_forecast = self.model.predict(X)
        # Approximate confidence intervals using quantile regression
        # retrained models or residual-based estimation
        residuals = self._compute_residuals(X)
        lower = point_forecast - 1.645 * residuals.std()
        upper = point_forecast + 1.645 * residuals.std()
        return {
            "quantity": point_forecast,
            "confidence_lower": np.maximum(lower, 0),
            "confidence_upper": upper,
        }

    def feature_importance(self) -> pd.DataFrame:
        """Return feature importances for explainability."""
        return pd.DataFrame({
            "feature": self.model.feature_name(),
            "importance": self.model.feature_importance(importance_type="gain"),
        }).sort_values("importance", ascending=False)
```

**Testing:**
- Model trains on 1 year of daily demand data for a single product, completes in under 10 seconds
- Predictions are non-negative (demand cannot be negative)
- MAPE on holdout set (last 30 days) is below 25% for smooth demand patterns
- Confidence intervals: `confidence_lower <= quantity <= confidence_upper` for all predictions
- Feature importance returns a DataFrame with all feature names and non-negative importance scores
- Model serialization: trained model can be saved to disk (joblib) and reloaded with identical predictions
- TimeSeriesSplit validation: no future data leakage -- training data always precedes validation data chronologically
- Multi-product training: model trained per product or globally with product_id as a feature; both modes produce forecasts

### Task 2.3: Forecast Generation Pipeline

**What:** Orchestrate end-to-end forecast generation: load demand history, engineer features, train model, generate forecasts, compute accuracy metrics, store results.

**Design:**
```python
# backend/app/ml/pipeline.py
class ForecastPipeline:
    async def run(self, tenant_id: UUID, config: ForecastConfig) -> ForecastGeneration:
        """
        Full pipeline: data load -> feature eng -> train -> predict -> evaluate -> store.
        Creates a forecast_generation record linking all forecast rows.
        """
        # 1. Load demand history
        history = await self.load_demand_history(tenant_id, config.lookback_days)

        # 2. Feature engineering
        features = build_features(history)

        # 3. Train/test split (temporal)
        X_train, X_val, y_train, y_val = temporal_split(features, config.validation_days)

        # 4. Train model
        model = LightGBMForecaster(config.model_params)
        train_metrics = model.train(X_train, y_train, X_val, y_val)

        # 5. Generate forecasts for next N days
        forecast_features = self.build_forecast_features(features, config.horizon_days)
        predictions = model.predict(forecast_features)

        # 6. Evaluate accuracy on validation period
        val_predictions = model.predict(X_val)
        accuracy = evaluate_forecast(y_val, val_predictions["quantity"])

        # 7. Store results
        generation = await self.store_forecast(
            tenant_id, predictions, accuracy, config, model
        )
        return generation
```

**Testing:**
- Full pipeline runs end-to-end: given 365 days of demand history, produces 90-day forecast
- `forecast_generation` record created with `status=completed`, `overall_mape` populated
- `demand_forecast` rows created for each `(product_id, location_id, forecast_date)` combination
- `generation_id` links all forecast rows to the generation record
- Pipeline handles products with zero demand history gracefully (skips with warning, not crash)
- Pipeline handles products with sparse/intermittent demand (many zeros) without erroring
- Concurrent pipeline runs for same tenant are serialized (no duplicate generation_ids)
- Pipeline runtime: < 2 minutes for 500 products x 8 locations x 90 days

### Task 2.4: Forecast Override Workflow

**What:** Allow planners to adjust AI-generated forecasts with reason codes, approval workflow, and audit trail.

**Design:**
```python
# backend/app/api/v1/forecasts.py
@router.post("/{forecast_id}/override", response_model=ForecastResponse)
async def override_forecast(
    forecast_id: UUID,
    override: ForecastOverrideRequest,  # {override_qty, reason, override_type}
    db: AsyncSession = Depends(get_db),
    current_user: AppUser = Depends(require_role(["planner", "admin"])),
):
    """
    Override a forecast point. Sets is_overridden=true, stores override_qty,
    override_reason, override_by. Writes audit_log entry.
    """
    forecast = await get_forecast_or_404(db, forecast_id)
    forecast.is_overridden = True
    forecast.override_qty = override.override_qty
    forecast.override_reason = override.reason
    forecast.override_by = current_user.id
    await write_audit_log(db, "demand_forecast", forecast_id, "override", {
        "original_qty": float(forecast.quantity),
        "override_qty": float(override.override_qty),
        "reason": override.reason,
    })
    await db.commit()
    return forecast
```

**Testing:**
- Planner overrides forecast from 450 to 520 with reason "Q3 promotion": `is_overridden=true`, `override_qty=520`
- Audit log contains entry with `action=override`, old/new values
- `override_type` accepts valid values: `promotion`, `market_intelligence`, `executive_override`, `other`
- Viewer role cannot override (403 Forbidden)
- Override of already-overridden forecast updates values (not creates duplicate)
- GET forecasts endpoint returns effective quantity: `override_qty` if overridden, else `quantity`
- Override history: multiple overrides on same forecast tracked in audit_log with timestamps

### Task 2.5: Forecast Accuracy Dashboard

**What:** Frontend page displaying forecast vs. actuals comparison, MAPE/WMAPE metrics, accuracy trends, and confidence interval visualization.

**Design:**
```typescript
// frontend/src/pages/ForecastsPage.tsx
// - Time series chart: actual demand (solid line) vs forecast (dashed line) with
//   confidence interval shading (P10-P90)
// - KPI cards: MAPE, WMAPE, Forecast Bias, Coverage (% within CI)
// - Accuracy trend: MAPE by week over last 12 weeks
// - Product selector: dropdown/search to filter by product + location
// - Generation selector: compare current vs previous forecast generations
// - Override indicator: points where planner overrode are marked differently

// frontend/src/components/forecasts/ForecastChart.tsx
import { LineChart, Line, Area, XAxis, YAxis, Tooltip, ResponsiveContainer } from "recharts";

interface ForecastChartProps {
  data: Array<{
    date: string;
    actual: number | null;
    forecast: number;
    confidence_lower: number;
    confidence_upper: number;
    is_overridden: boolean;
  }>;
}
```

**Testing:**
- Chart renders with actual demand line and forecast line
- Confidence interval renders as shaded area between P10 and P90
- MAPE card shows correct value matching backend calculation
- Product selector filters chart data to selected product + location
- Overridden forecast points display with distinct marker (e.g., orange dot)
- Chart handles missing actuals gracefully (gap in actual line for future dates)
- Generation comparison: selecting two generations overlays both forecast lines
- Responsive: chart resizes correctly on window resize

### Definition of Done -- Phase 2
- [ ] LightGBM model trains on demand history and produces forecasts with confidence intervals
- [ ] Feature engineering produces correct lag, rolling, and calendar features
- [ ] Forecast pipeline runs end-to-end from history to stored forecast rows
- [ ] MAPE/WMAPE accuracy metrics computed and stored on each generation
- [ ] Override workflow allows planners to adjust forecasts with audit trail
- [ ] Frontend forecast chart displays actuals vs forecast with confidence intervals
- [ ] KPI dashboard shows MAPE, WMAPE, bias, and accuracy trends
- [ ] API endpoint: POST `/forecasts/generate` triggers async pipeline
- [ ] API endpoint: GET `/forecasts?product_id=X&generation_id=Y` returns forecast data
- [ ] Model accuracy on test datasets: MAPE < 25% for smooth demand, < 40% for intermittent
- [ ] Test suite: all ML unit tests and integration tests pass

---

## Phase 3: Inventory Planning & Scenario Modeling

**Duration:** 3-4 weeks
**Dependencies:** Phase 2 (demand forecasts exist to drive inventory calculations)
**Goal:** Safety stock calculation, reorder point optimization, multi-location inventory visibility, and what-if scenario modeling for demand shocks and supply disruptions.

### Task 3.1: Inventory Visibility Dashboard

**What:** Multi-location inventory position view showing on-hand, allocated, in-transit, and available quantities with color-coded health indicators.

**Design:**
```python
# backend/app/api/v1/inventory.py
@router.get("/positions", response_model=InventoryPositionList)
async def list_inventory_positions(
    location_id: UUID | None = None,
    product_id: UUID | None = None,
    health_status: str | None = None,  # overstock, healthy, low, stockout
    page: int = Query(1, ge=1),
    page_size: int = Query(50, ge=1, le=500),
    ...
):
    """
    Return inventory positions with computed health status.
    Health = comparison of available_qty vs safety_stock and reorder_point
    from inventory_policy.
    """
    # Join inventory_level with inventory_policy to compute health status
    ...
```

```typescript
// frontend/src/pages/InventoryPage.tsx
// - Data grid: Product | Location | On Hand | Allocated | In Transit | Available | Safety Stock | Health
// - Health column: color-coded badge (red=stockout, orange=low, green=healthy, blue=overstock)
// - Filters: location dropdown, health status filter, product search
// - Summary cards: Total SKU count, Stockout count, Low stock count, Inventory value ($)
```

**Testing:**
- Grid displays correct quantities from `inventory_level` table
- Health status computed correctly: `available_qty = 0` -> stockout; `available_qty < safety_stock` -> low; `available_qty > max_stock` -> overstock; else healthy
- Filter by health status `stockout` returns only stockout items
- Filter by location shows only inventory at that location
- Summary cards show correct aggregate counts
- Empty inventory (no records) shows "No inventory data. Import data to get started."

### Task 3.2: Safety Stock & Reorder Point Calculation

**What:** Compute statistical safety stock and reorder points from demand variability and lead time, store in `inventory_policy`, allow planner override.

**Design:**
```python
# backend/app/services/inventory_service.py
def calculate_safety_stock(
    demand_history: pd.Series,
    lead_time_days: int,
    lead_time_std_dev: float,
    service_level: float = 0.95,
) -> dict:
    """
    Statistical safety stock using demand and lead time variability.
    SS = Z * sqrt(LT * sigma_d^2 + d_avg^2 * sigma_LT^2)
    where Z = norm.ppf(service_level), LT = avg lead time,
    sigma_d = demand std dev, d_avg = avg daily demand, sigma_LT = lead time std dev.
    """
    from scipy.stats import norm
    z = norm.ppf(service_level)
    avg_demand = demand_history.mean()
    std_demand = demand_history.std()
    safety_stock = z * np.sqrt(
        lead_time_days * std_demand**2 + avg_demand**2 * lead_time_std_dev**2
    )
    reorder_point = avg_demand * lead_time_days + safety_stock
    reorder_qty = avg_demand * lead_time_days * 2  # Simple EOQ approximation
    return {
        "safety_stock": round(safety_stock, 2),
        "reorder_point": round(reorder_point, 2),
        "reorder_qty": round(reorder_qty, 2),
        "service_level_target": service_level,
        "calculation_method": "statistical",
        "parameters": {
            "z_score": round(z, 4),
            "demand_std_dev": round(std_demand, 2),
            "avg_daily_demand": round(avg_demand, 2),
            "lead_time_std_dev": round(lead_time_std_dev, 2),
        },
    }

@router.post("/inventory/policies/recalculate")
async def recalculate_policies(config: PolicyRecalcRequest, ...):
    """Recalculate safety stock for all product-location pairs using latest demand history."""
    ...
```

**Testing:**
- Safety stock for product with daily demand mean=100, std=20, lead time=14 days, LT std=2 days, 95% service level: verify against hand-calculated value
- Reorder point = avg_demand * lead_time + safety_stock (verify math)
- Service level 0.99 produces higher safety stock than 0.95 (same inputs)
- Zero demand variability (std=0) produces safety stock of 0
- Results stored in `inventory_policy` with correct `parameters` JSONB
- Recalculation endpoint updates existing policies, not creates duplicates

### Task 3.3: What-If Scenario Modeling

**What:** Allow planners to create scenarios that modify demand forecasts or supply parameters, then see the impact on inventory positions and stockout risk.

**Design:**
```python
# backend/app/api/v1/scenarios.py
@router.post("/scenarios", response_model=ScenarioResponse)
async def create_scenario(scenario: ScenarioCreate, ...):
    """
    Create a what-if scenario. Types:
    - demand_shock: multiply demand by a factor for a date range
    - supply_disruption: delay supplier deliveries by N days
    - lead_time_change: modify lead time for a supplier/product
    """
    ...

@router.post("/scenarios/{scenario_id}/simulate", response_model=SimulationResult)
async def run_simulation(scenario_id: UUID, ...):
    """
    Run inventory simulation with modified parameters.
    Returns projected inventory levels, stockout dates, and fill rate impact.
    Does NOT modify actual data -- simulation results stored in JSONB.
    """
    scenario = await get_scenario_or_404(db, scenario_id)
    # Load base forecast and inventory
    # Apply scenario modifications (demand multiplier, lead time shift)
    # Simulate daily inventory evolution over horizon
    simulation = simulate_inventory(base_forecast, base_inventory, scenario.modifications)
    return SimulationResult(
        scenario_id=scenario_id,
        projected_stockout_dates=simulation.stockout_dates,
        projected_fill_rate=simulation.fill_rate,
        projected_inventory=simulation.daily_positions,
        comparison_to_base={
            "fill_rate_delta": simulation.fill_rate - base_fill_rate,
            "stockout_days_delta": simulation.stockout_days - base_stockout_days,
        },
    )
```

**Testing:**
- Demand shock scenario: 30% demand increase for next 30 days shows reduced fill rate and earlier stockout dates
- Supply disruption scenario: 14-day delay on primary supplier shows inventory dropping below safety stock
- Simulation does not modify actual forecast or inventory data (read-only)
- Scenario comparison: side-by-side view of base case vs scenario shows delta in KPIs
- Multiple scenarios can be created and compared simultaneously
- Invalid scenario (e.g., negative demand multiplier) returns 400 with validation error

### Task 3.4: Replenishment Recommendations

**What:** Generate purchase order recommendations when projected inventory drops below reorder point.

**Design:**
```python
# backend/app/services/inventory_service.py
async def generate_replenishment_recommendations(tenant_id: UUID, horizon_days: int = 30):
    """
    For each product-location pair:
    1. Project inventory forward using forecast and in-transit POs
    2. If projected available < reorder_point, recommend purchase
    3. Select preferred supplier, recommended quantity, and target date
    """
    recommendations = []
    for product_id, location_id in product_locations:
        projected = project_inventory(product_id, location_id, horizon_days)
        policy = get_inventory_policy(product_id, location_id)
        for day, position in projected.items():
            if position < policy.reorder_point:
                rec = ReplenishmentRecommendation(
                    product_id=product_id,
                    location_id=location_id,
                    supplier_id=get_preferred_supplier(product_id),
                    recommended_qty=policy.reorder_qty,
                    recommended_date=day - timedelta(days=supplier_lead_time),
                    urgency="critical" if position <= 0 else "urgent" if position < policy.safety_stock else "normal",
                    reason=f"Projected stockout on {day}",
                )
                recommendations.append(rec)
                break  # Only one recommendation per product-location
    return recommendations
```

**Testing:**
- Product with projected stockout in 10 days generates recommendation with `urgency=critical`
- Product above reorder point generates no recommendation
- Recommended date accounts for supplier lead time (order must be placed lead_time_days before stockout)
- Recommended quantity equals `reorder_qty` from inventory policy
- Preferred supplier selected when multiple suppliers available
- Recommendations persisted in `replenishment_recommendation` table with `status=pending`
- Planner can approve/reject recommendations via API (status transitions)

### Definition of Done -- Phase 3
- [ ] Inventory visibility grid shows all product-location positions with health indicators
- [ ] Safety stock calculation matches statistical formula for all test cases
- [ ] Inventory policies stored and updatable per product-location pair
- [ ] What-if scenarios simulate demand shocks and supply disruptions without modifying real data
- [ ] Scenario comparison shows KPI deltas (fill rate, stockout days)
- [ ] Replenishment recommendations generated when projected inventory < reorder point
- [ ] Frontend inventory page with filters, health badges, and summary cards
- [ ] Frontend scenario builder UI with parameter inputs and result visualization
- [ ] Test suite: inventory math validated against hand-calculated expected values

---

## Phase 4: Advanced ML & Explainability

**Duration:** 4-5 weeks
**Dependencies:** Phase 2 (LightGBM baseline exists for comparison)
**Goal:** Temporal Fusion Transformer model, SHAP-based explainability, external signal integration (weather, holidays), and model ensembling.

### Task 4.1: Temporal Fusion Transformer (TFT) Model

**What:** Implement PyTorch-based TFT for multi-horizon probabilistic demand forecasting.

**Design:**
```python
# backend/app/ml/models/tft_model.py
import pytorch_lightning as pl
from pytorch_forecasting import TemporalFusionTransformer, TimeSeriesDataSet

class TFTForecaster:
    def __init__(self, config: TFTConfig):
        self.config = config

    def prepare_dataset(self, demand_df: pd.DataFrame) -> TimeSeriesDataSet:
        """Prepare pytorch-forecasting TimeSeriesDataSet from demand history."""
        return TimeSeriesDataSet(
            demand_df,
            time_idx="time_idx",
            target="quantity",
            group_ids=["product_id", "location_id"],
            max_encoder_length=self.config.encoder_length,  # lookback
            max_prediction_length=self.config.prediction_length,  # horizon
            time_varying_known_reals=["day_of_week", "month", "is_holiday"],
            time_varying_unknown_reals=["quantity"],
            static_categoricals=["product_id", "location_id"],
        )

    def train(self, train_ds: TimeSeriesDataSet, val_ds: TimeSeriesDataSet) -> dict:
        """Train TFT model with early stopping."""
        model = TemporalFusionTransformer.from_dataset(
            train_ds,
            learning_rate=self.config.learning_rate,
            hidden_size=self.config.hidden_size,
            attention_head_size=self.config.attention_heads,
            dropout=self.config.dropout,
            loss=QuantileLoss(),  # native probabilistic output
        )
        trainer = pl.Trainer(
            max_epochs=self.config.max_epochs,
            callbacks=[EarlyStopping(monitor="val_loss", patience=5)],
            accelerator="auto",  # GPU if available
        )
        trainer.fit(model, train_dataloaders=train_dl, val_dataloaders=val_dl)
        return {"best_epoch": trainer.current_epoch}

    def predict(self, dataset: TimeSeriesDataSet) -> dict:
        """Generate quantile forecasts (P10, P50, P90)."""
        predictions = self.model.predict(dataset, mode="quantiles")
        return {
            "quantity": predictions[:, :, 1],      # P50 (median)
            "confidence_lower": predictions[:, :, 0],  # P10
            "confidence_upper": predictions[:, :, 2],  # P90
        }
```

**Testing:**
- TFT model trains on 2 years of daily demand data for 50 products, completes within 30 minutes on CPU (5 minutes on GPU)
- Quantile predictions: P10 <= P50 <= P90 for all forecast points
- MAPE improvement: TFT MAPE <= LightGBM MAPE on holdout set (at least not worse)
- Model handles products with intermittent demand (many zero values) without NaN predictions
- Attention weights extractable for explainability (variable selection network outputs)
- Model checkpoint saved and reloadable for inference without retraining

### Task 4.2: SHAP Explainability Layer

**What:** Compute SHAP values for each forecast point to explain which features drive the prediction.

**Design:**
```python
# backend/app/ml/explainability.py
import shap

class ForecastExplainer:
    def __init__(self, model, model_type: str):
        self.model_type = model_type
        if model_type == "lightgbm":
            self.explainer = shap.TreeExplainer(model)
        elif model_type == "tft":
            # TFT has built-in attention-based variable importance
            self.explainer = None
            self.tft_model = model

    def explain(self, X: pd.DataFrame, top_k: int = 5) -> list[dict]:
        """
        Return top-K feature drivers for each prediction row.
        Output: [{"feature": "lag_7d", "shap_value": 0.23, "direction": "positive"}, ...]
        """
        if self.model_type == "lightgbm":
            shap_values = self.explainer.shap_values(X)
            results = []
            for i in range(len(X)):
                row_shap = pd.Series(shap_values[i], index=X.columns)
                top = row_shap.abs().nlargest(top_k)
                drivers = [
                    {"feature": feat, "shap_value": round(float(row_shap[feat]), 4),
                     "direction": "positive" if row_shap[feat] > 0 else "negative"}
                    for feat in top.index
                ]
                results.append(drivers)
            return results
        elif self.model_type == "tft":
            # Use TFT's variable selection network attention weights
            interpretation = self.tft_model.interpret_output(predictions)
            return self._format_tft_attention(interpretation, top_k)
```

**Testing:**
- SHAP values sum to (prediction - base_value) for each row (SHAP additivity property)
- Top-5 drivers returned for each forecast point
- Direction field is "positive" for features increasing forecast, "negative" for decreasing
- SHAP computation for 100 forecast points completes in under 30 seconds
- SHAP values stored in `demand_forecast.model_details` JSONB as `top_drivers` array
- TFT attention-based importance returns variable names and attention weights

### Task 4.3: External Signal Integration

**What:** Ingest external data signals (public holidays, weather forecasts) as additional features for ML models.

**Design:**
```python
# backend/app/ml/external_signals.py
class ExternalSignalProvider:
    async def get_holidays(self, country_code: str, year: int) -> pd.DataFrame:
        """Fetch public holidays from API. Returns date + holiday_name."""
        # Use `holidays` Python library for offline holiday calendars
        ...

    async def get_weather(self, latitude: float, longitude: float,
                          start_date: date, end_date: date) -> pd.DataFrame:
        """Fetch historical/forecast weather from Open-Meteo API (free, no key)."""
        # Columns: date, temp_max, temp_min, precipitation_mm, wind_speed
        ...

    def merge_signals(self, demand_df: pd.DataFrame,
                      signals: dict[str, pd.DataFrame]) -> pd.DataFrame:
        """Left-join external signals onto demand DataFrame by date."""
        result = demand_df.copy()
        for signal_name, signal_df in signals.items():
            result = result.merge(signal_df, on="date", how="left")
        return result
```

**Testing:**
- Holiday calendar for US 2026 includes July 4, Thanksgiving, Christmas
- Weather data from Open-Meteo returns temperature and precipitation for given lat/long
- Merged DataFrame has correct signal columns joined by date
- Missing signal data (e.g., weather API down) fills with NaN, model handles gracefully
- Signal features improve MAPE by measurable amount on seasonal products (e.g., ice cream)

### Task 4.4: Model Ensembling

**What:** Combine LightGBM and TFT predictions using weighted average or stacking.

**Design:**
```python
# backend/app/ml/models/ensemble.py
class EnsembleForecaster:
    def __init__(self, models: list[tuple[str, object]], weights: list[float] | None = None):
        self.models = models  # [("lightgbm", lgb_model), ("tft", tft_model)]
        self.weights = weights or [1/len(models)] * len(models)

    def predict(self, X_lgb: pd.DataFrame, X_tft: TimeSeriesDataSet) -> dict:
        predictions = []
        for (name, model), weight in zip(self.models, self.weights):
            if name == "lightgbm":
                pred = model.predict(X_lgb)
            elif name == "tft":
                pred = model.predict(X_tft)
            predictions.append(pred["quantity"] * weight)
        ensemble_qty = sum(predictions)
        return {"quantity": ensemble_qty, "model_type": "ensemble"}
```

**Testing:**
- Ensemble of equally weighted LightGBM + TFT produces average of individual predictions
- Ensemble MAPE <= min(individual model MAPEs) in at least 60% of test cases
- Weights of [0.7, 0.3] correctly apply 70% LightGBM + 30% TFT
- Ensemble generation stores `model_type=ensemble` in forecast_generation record

### Definition of Done -- Phase 4
- [ ] TFT model trains and produces probabilistic forecasts (P10/P50/P90)
- [ ] SHAP values computed for LightGBM; attention weights extracted for TFT
- [ ] Explainability UI shows top drivers per forecast point with bar chart
- [ ] External signals (holidays, weather) integrated into feature pipeline
- [ ] Ensemble model combines LightGBM + TFT predictions
- [ ] Model comparison dashboard: MAPE/WMAPE by model type per product
- [ ] Forecast generation supports model selection: `lightgbm`, `tft`, `ensemble`
- [ ] GPU training path works when CUDA available; falls back to CPU gracefully
- [ ] Test suite: ML model tests pass with deterministic seeds

---

## Phase 5: Supplier Performance & Structured Risk Scoring

**Duration:** 3-4 weeks
**Dependencies:** Phase 2 (demand data exists); can run in parallel with Phase 3/4
**Goal:** Supplier performance dashboards, structured risk scoring from operational data, and basic ERP connector.

### Task 5.1: Supplier Performance Metrics

**What:** Compute supplier KPIs from purchase order data: OTIF rate, average lead time, lead time variability, defect rate.

**Design:**
```python
# backend/app/services/supplier_service.py
async def calculate_supplier_performance(
    tenant_id: UUID, supplier_id: UUID, period_start: date, period_end: date
) -> SupplierPerformance:
    """
    Compute performance metrics from purchase_order + purchase_order_line data.
    """
    pos = await get_completed_pos(tenant_id, supplier_id, period_start, period_end)
    on_time = sum(1 for po in pos if po.actual_delivery_date <= po.expected_delivery_date)
    in_full = sum(1 for po in pos if all(
        line.received_qty >= line.quantity for line in po.lines
    ))
    otif = sum(1 for po in pos if (
        po.actual_delivery_date <= po.expected_delivery_date and
        all(line.received_qty >= line.quantity for line in po.lines)
    ))
    lead_times = [(po.actual_delivery_date - po.order_date).days for po in pos]
    defect_qty = sum(line.rejected_qty for po in pos for line in po.lines)
    total_qty = sum(line.received_qty + line.rejected_qty for po in pos for line in po.lines)

    return SupplierPerformance(
        on_time_delivery_rate=on_time / len(pos) if pos else None,
        in_full_rate=in_full / len(pos) if pos else None,
        otif_rate=otif / len(pos) if pos else None,
        avg_lead_time_days=np.mean(lead_times) if lead_times else None,
        lead_time_std_dev=np.std(lead_times) if lead_times else None,
        defect_rate=defect_qty / total_qty if total_qty > 0 else None,
        total_orders=len(pos),
    )
```

**Testing:**
- Supplier with 10 POs, 8 on-time: `on_time_delivery_rate = 0.80`
- OTIF requires BOTH on-time AND in-full: PO on-time but short-shipped is not OTIF
- Lead time calculated as `actual_delivery_date - order_date` in days
- Defect rate: 5 rejected out of 1000 received = 0.005
- No completed POs returns null metrics (not zero)
- Performance stored as time-series in `supplier_risk_assessment` table

### Task 5.2: Structured Risk Scoring

**What:** Compute composite supplier risk score from operational performance, geographic concentration, and financial indicators.

**Design:**
```python
# backend/app/services/risk_service.py
class StructuredRiskScorer:
    def score(self, supplier_id: UUID, performance: SupplierPerformance,
              supplier: Supplier) -> SupplierRiskAssessment:
        scores = {}
        # Operational risk (0-100, higher = more risk)
        scores["operational"] = self._operational_score(performance)
        # Geographic risk based on country
        scores["geopolitical"] = self._geopolitical_score(supplier.country_code)
        # Concentration risk (how dependent are we on this supplier?)
        scores["concentration"] = self._concentration_score(supplier_id)
        # Financial (placeholder until unstructured data in Phase 7)
        scores["financial"] = 50  # neutral default

        overall = sum(
            score * weight for score, weight in
            zip(scores.values(), [0.35, 0.25, 0.25, 0.15])
        )
        risk_tier = "critical" if overall > 80 else "high" if overall > 60 else "medium" if overall > 40 else "low"
        return SupplierRiskAssessment(
            overall_score=overall, risk_tier=risk_tier, scores=scores
        )

    def _operational_score(self, perf: SupplierPerformance) -> float:
        """Low OTIF = high risk. Map OTIF 0.5-1.0 to risk 100-0."""
        if perf.otif_rate is None:
            return 50  # unknown = medium risk
        return max(0, min(100, (1 - perf.otif_rate) * 200))
```

**Testing:**
- Supplier with OTIF=0.95 gets low operational risk score (~10)
- Supplier with OTIF=0.60 gets high operational risk score (~80)
- Country in sanctioned/high-risk list gets high geopolitical score
- Concentration score increases when supplier provides >50% of spend for any product
- Overall score is weighted average of component scores
- Risk tier thresholds: <=40=low, 41-60=medium, 61-80=high, >80=critical
- Risk assessment stored with `scores` JSONB containing all component scores

### Task 5.3: Supplier Performance Dashboard

**What:** Frontend dashboard showing supplier scorecard, performance trends, risk tier distribution, and drilldown to individual suppliers.

**Design:**
```typescript
// frontend/src/pages/SuppliersPage.tsx
// - Supplier list with columns: Name | Country | OTIF (90d) | Lead Time | Risk Tier | Badge
// - Risk tier distribution: pie chart showing count of low/medium/high/critical suppliers
// - Supplier detail page:
//   - Performance trend: OTIF rate over last 12 months (line chart)
//   - Lead time trend: avg and std dev over time
//   - Risk score breakdown: radar chart of operational/geopolitical/concentration/financial
//   - Active POs: list of open purchase orders with status
```

**Testing:**
- Supplier list sorts by risk tier (critical first) by default
- Risk tier badges use correct colors: green=low, yellow=medium, orange=high, red=critical
- Clicking supplier navigates to detail page with correct data
- OTIF trend chart shows 12 monthly data points
- Radar chart renders 4 risk dimension axes

### Task 5.4: REST API Data Connector

**What:** Enable external systems to push demand history, inventory levels, and PO data via authenticated REST API (in addition to CSV import).

**Design:**
```python
# backend/app/api/v1/imports.py
@router.post("/api/v1/demand-history/batch", status_code=201)
async def batch_import_demand_history(
    records: list[DemandHistoryRecord],  # max 10,000 per request
    db: AsyncSession = Depends(get_db),
    current_user: AppUser = Depends(require_role(["admin", "api"])),
):
    """Bulk upsert demand history records. Idempotent on (product_sku, location_code, demand_date)."""
    ...

@router.post("/api/v1/inventory/sync", status_code=200)
async def sync_inventory_levels(
    positions: list[InventorySync],
    db: AsyncSession = Depends(get_db),
    current_user: AppUser = Depends(require_role(["admin", "api"])),
):
    """Full or delta inventory sync from external ERP."""
    ...
```

**Testing:**
- Batch import of 5,000 demand history records completes in < 5 seconds
- Duplicate records (same product/location/date) upsert without error
- Invalid product SKU returns row-level error without failing entire batch
- API key authentication works for machine-to-machine calls
- Rate limiting: 100 requests/minute per API key
- Request body > 10,000 records returns 400 with "batch too large" error

### Definition of Done -- Phase 5
- [ ] Supplier OTIF, lead time, and defect rate computed from PO data
- [ ] Structured risk scoring produces composite score with component breakdown
- [ ] Supplier dashboard shows performance trends, risk distribution, and drilldown
- [ ] REST API batch endpoints for demand history, inventory, and PO data
- [ ] API key authentication for machine-to-machine integration
- [ ] Rate limiting on API endpoints
- [ ] Test suite: risk scoring math validated against hand-calculated values

---

## Phase 6: Graph Layer & Network Analysis

**Duration:** 3-4 weeks
**Dependencies:** Phase 5 (supplier data exists for graph population)
**Goal:** Add graph_node/graph_edge tables, populate supply chain network graph, enable multi-tier supplier analysis and risk propagation queries.

### Task 6.1: Graph Schema & Sync Triggers

**What:** Create `graph_node` and `graph_edge` tables. Build triggers to sync master data changes to the graph layer.

**Design:**
```python
# backend/app/models/graph.py
class GraphNode(Base):
    __tablename__ = "graph_node"
    id = Column(UUID, primary_key=True, server_default=text("gen_random_uuid()"))
    tenant_id = Column(UUID, nullable=False)
    node_type = Column(String(50), nullable=False)  # supplier, product, location, customer
    entity_id = Column(UUID, nullable=False)
    label = Column(String(500), nullable=False)
    properties = Column(JSONB, nullable=False, server_default=text("'{}'::jsonb"))
    __table_args__ = (UniqueConstraint("tenant_id", "node_type", "entity_id"),)

class GraphEdge(Base):
    __tablename__ = "graph_edge"
    id = Column(UUID, primary_key=True, server_default=text("gen_random_uuid()"))
    tenant_id = Column(UUID, nullable=False)
    source_node_id = Column(UUID, ForeignKey("graph_node.id"), nullable=False)
    target_node_id = Column(UUID, ForeignKey("graph_node.id"), nullable=False)
    edge_type = Column(String(100), nullable=False)  # SUPPLIES, SHIPS_TO, COMPONENT_OF, etc.
    properties = Column(JSONB, nullable=False, server_default=text("'{}'::jsonb"))
    weight = Column(Numeric(10, 4), server_default=text("1.0"))
    is_active = Column(Boolean, nullable=False, server_default=text("true"))
    valid_from = Column(TIMESTAMP(timezone=True), server_default=text("now()"))
    valid_to = Column(TIMESTAMP(timezone=True))

# PostgreSQL trigger to sync product/supplier/location creates to graph_node
# (see Data Model Suggestion 4 for trigger implementation)
```

**Testing:**
- Creating a product in relational table auto-creates corresponding `graph_node` with `node_type=product`
- Updating product name updates `graph_node.label`
- Deleting (soft-delete) product marks associated edges as `is_active=false`
- `graph_node` unique constraint prevents duplicate nodes for same entity
- Edge between non-existent nodes fails with FK violation

### Task 6.2: Supply Network Population

**What:** Populate graph edges from existing data: supplier-product relationships from PO history, location-to-location transport links, BOM relationships.

**Design:**
```python
# backend/app/services/graph_service.py
async def populate_supply_network(tenant_id: UUID):
    """
    Build graph edges from existing relational data:
    1. SUPPLIES edges from purchase_order_line (which suppliers supply which products)
    2. SHIPS_TO edges from purchase_order (supplier location -> ship-to location)
    3. STORES_AT edges from inventory_level (which products at which locations)
    """
    # Derive SUPPLIES edges from PO history
    supplier_products = await db.execute(text("""
        SELECT DISTINCT po.supplier_id, pol.product_id,
               AVG((po.actual_delivery_date - po.order_date)) AS avg_lead_time,
               AVG(pol.unit_cost) AS avg_cost
        FROM purchase_order po
        JOIN purchase_order_line pol ON pol.purchase_order_id = po.id
        WHERE po.tenant_id = :tenant_id AND po.status = 'received'
        GROUP BY po.supplier_id, pol.product_id
    """))
    for row in supplier_products:
        await create_or_update_edge(
            tenant_id, "supplier", row.supplier_id,
            "product", row.product_id,
            "SUPPLIES",
            {"avg_lead_time_days": row.avg_lead_time, "avg_unit_cost": float(row.avg_cost)},
        )
```

**Testing:**
- After population, supplier with 3 distinct products in PO history has 3 SUPPLIES edges
- Edge properties contain correct avg lead time and cost from PO data
- STORES_AT edges created for each non-zero inventory_level record
- Re-running population is idempotent (upserts edges, not duplicates)
- Tenant isolation: graph data scoped by tenant_id

### Task 6.3: Network Analysis API

**What:** API endpoints for multi-tier supplier tracing, risk propagation, alternative supplier discovery, and concentration risk detection.

**Design:**
```python
# backend/app/api/v1/graph.py
@router.get("/graph/supplier-network/{product_id}")
async def get_supplier_network(product_id: UUID, max_depth: int = 3, ...):
    """
    Return full supplier network for a product (Tier 1, 2, 3+).
    Uses recursive CTE on graph_edge table.
    """
    ...

@router.get("/graph/impact-analysis/{supplier_id}")
async def analyze_supplier_impact(supplier_id: UUID, ...):
    """
    Risk propagation: which products, locations, and customers are affected
    if this supplier is disrupted?
    """
    ...

@router.get("/graph/concentration-risk")
async def get_concentration_risks(threshold: float = 0.8, ...):
    """
    Find products dependent on single geographic region for >threshold of supply.
    """
    ...

@router.get("/graph/alternative-suppliers/{product_id}")
async def find_alternatives(product_id: UUID, exclude_supplier_id: UUID, ...):
    """
    Find alternative suppliers for a product, excluding the disrupted one.
    Returns qualified alternatives with cost premium and lead time data.
    """
    ...
```

**Testing:**
- Supplier network for product with 2 Tier-1 and 3 Tier-2 suppliers returns 5 nodes in correct tiers
- Impact analysis for critical supplier returns all downstream products and locations
- Concentration risk detects product where 90% of supply comes from one country
- Alternative supplier query excludes the disrupted supplier and returns only active alternatives
- Recursive queries terminate at max_depth (no infinite loops on cyclic graphs)
- Graph queries complete in < 2 seconds for networks with 1,000 nodes and 5,000 edges

### Task 6.4: Supply Network Visualization

**What:** Interactive graph visualization in the frontend showing supplier network topology, risk propagation paths, and concentration hotspots.

**Design:**
```typescript
// frontend/src/components/graph/NetworkGraph.tsx
// - Force-directed graph layout using react-force-graph or vis-network
// - Node types: color-coded (supplier=blue, product=green, location=orange, customer=purple)
// - Edge types: styled by type (SUPPLIES=solid, SUB_SUPPLIER_OF=dashed, RISK_EXPOSURE=red)
// - Click node: show detail panel with entity info and connected edges
// - Risk propagation mode: highlight all downstream nodes affected by selected supplier
// - Concentration risk overlay: heat-map coloring by geographic concentration
```

**Testing:**
- Graph renders with correct node count and edge count from API response
- Node colors match type legend
- Clicking a supplier node shows connected products and performance metrics
- Risk propagation highlight activates when "Impact Analysis" button clicked
- Graph handles 500+ nodes without performance degradation (canvas rendering, not SVG)

### Definition of Done -- Phase 6
- [ ] `graph_node` and `graph_edge` tables created with sync triggers
- [ ] Supply network populated from PO history and inventory data
- [ ] Multi-tier supplier network query returns correct tiers (tested with known topology)
- [ ] Impact analysis correctly identifies all downstream entities
- [ ] Concentration risk query detects single-source dependencies
- [ ] Network visualization renders interactive graph with node/edge detail
- [ ] Graph queries perform within 2-second SLA for 1,000-node networks
- [ ] Test suite: graph queries tested with known topologies and expected traversal results

---

## Phase 7: Agentic AI & Disruption Response

**Duration:** 4-5 weeks
**Dependencies:** Phase 5 (risk scoring), Phase 6 (graph for impact analysis)
**Goal:** LLM-powered supplier risk scoring from unstructured data, autonomous disruption detection, and AI-generated re-sourcing recommendations.

### Task 7.1: LLM-Enhanced Supplier Risk Scoring

**What:** Use LLM to parse supplier news feeds, ESG reports, and financial filings to produce unstructured risk signals.

**Design:**
```python
# backend/app/workers/risk_worker.py
class LLMRiskAnalyzer:
    def __init__(self, llm_client):
        self.llm = llm_client  # Anthropic or OpenAI client

    async def analyze_supplier_news(self, supplier_name: str, country: str) -> list[dict]:
        """
        Search news APIs for supplier mentions, parse with LLM for risk signals.
        Returns structured risk signals from unstructured sources.
        """
        articles = await self.fetch_news(supplier_name, days_back=30)
        signals = []
        for article in articles:
            prompt = f"""Analyze this news article for supply chain risk signals 
            related to {supplier_name}. Extract:
            - risk_type: one of [financial, operational, geopolitical, esg, labor, natural_disaster]
            - severity: one of [low, medium, high, critical]
            - summary: one-sentence summary of the risk
            - confidence: 0.0-1.0 confidence in the assessment

            Article: {article['text'][:2000]}
            
            Respond in JSON format."""
            response = await self.llm.analyze(prompt)
            signals.append({
                "source": article["source"],
                "date": article["date"],
                "headline": article["headline"],
                **response,
            })
        return signals

    async def analyze_esg_disclosure(self, document_text: str) -> dict:
        """Parse ESG/sustainability report for CSDDD compliance signals."""
        ...
```

**Testing:**
- News article about port strike produces `risk_type=operational`, `severity=high`
- Article about supplier bankruptcy produces `risk_type=financial`, `severity=critical`
- ESG report with good sustainability metrics produces positive ESG score
- LLM response parsed correctly into structured format
- Malformed LLM response (non-JSON) handled gracefully with fallback
- Rate limiting on LLM API calls (max 10 requests/minute per supplier)
- Risk signals stored in `supplier_risk_assessment.scores` JSONB under `llm_signals` key

### Task 7.2: Disruption Detection Agent

**What:** Background worker that monitors supplier risk signals and inventory projections to detect potential disruptions before they cause stockouts.

**Design:**
```python
# backend/app/workers/disruption_worker.py
class DisruptionDetector:
    async def run_detection_cycle(self, tenant_id: UUID):
        """
        Scheduled job (hourly) that checks for disruption signals:
        1. New high/critical risk signals from LLM analysis
        2. POs with delivery date past expected + threshold
        3. Inventory projections showing stockout within safety window
        """
        disruptions = []

        # Check for supplier risk escalations
        recent_risks = await self.get_recent_risk_changes(tenant_id, hours=1)
        for risk in recent_risks:
            if risk.severity in ("high", "critical"):
                disruptions.append(Disruption(
                    type="supplier_risk_escalation",
                    supplier_id=risk.supplier_id,
                    severity=risk.severity,
                    description=risk.summary,
                ))

        # Check for delayed POs
        delayed_pos = await self.get_delayed_pos(tenant_id, threshold_days=3)
        for po in delayed_pos:
            disruptions.append(Disruption(
                type="delivery_delay",
                supplier_id=po.supplier_id,
                severity="high" if po.delay_days > 7 else "medium",
                affected_po_ids=[po.id],
            ))

        # Generate response recommendations for each disruption
        for disruption in disruptions:
            recommendations = await self.generate_response(disruption)
            await self.store_disruption(disruption, recommendations)
```

**Testing:**
- Supplier risk escalation from medium to critical triggers disruption detection
- PO delayed by 5 days triggers delivery_delay disruption with `severity=medium`
- PO delayed by 10 days triggers delivery_delay disruption with `severity=high`
- Detection cycle completes in < 30 seconds for tenant with 100 suppliers
- Duplicate disruptions (same supplier, same type, same day) are deduplicated

### Task 7.3: AI-Generated Response Recommendations

**What:** Generate actionable recommendations (expedite, re-source, increase safety stock) for detected disruptions using LLM + graph data.

**Design:**
```python
# backend/app/workers/disruption_worker.py
async def generate_response(self, disruption: Disruption) -> list[Recommendation]:
    """
    Use graph analysis + LLM to generate response options:
    1. Graph: find alternative suppliers, assess impact scope
    2. LLM: generate natural language recommendation with cost/risk trade-offs
    """
    # Get impact analysis from graph
    impact = await graph_service.analyze_supplier_impact(disruption.supplier_id)
    alternatives = await graph_service.find_alternatives(disruption.supplier_id)

    # Generate recommendations
    recommendations = []
    if alternatives:
        for alt in alternatives[:3]:
            recommendations.append(Recommendation(
                action="re_source",
                description=f"Switch to {alt.name} ({alt.country_code}). "
                           f"Lead time: {alt.lead_time_days}d, cost premium: {alt.cost_premium_pct}%",
                cost_impact=self._estimate_cost_impact(alt),
                lead_time_days=alt.lead_time_days,
                confidence=0.85,
            ))
    recommendations.append(Recommendation(
        action="expedite",
        description="Request expedited shipping on affected POs",
        cost_impact=self._estimate_expedite_cost(disruption.affected_po_ids),
    ))
    recommendations.append(Recommendation(
        action="increase_safety_stock",
        description="Temporarily increase safety stock for affected products by 25%",
        cost_impact=self._estimate_inventory_holding_cost(),
    ))
    return recommendations
```

**Testing:**
- Disruption with available alternative supplier generates `re_source` recommendation
- Disruption with delayed PO generates `expedite` recommendation
- All disruptions generate `increase_safety_stock` recommendation as fallback
- Recommendations include cost impact estimates (not null)
- Recommendations stored with `status=pending` for planner review
- Planner can approve/reject each recommendation via API

### Task 7.4: Disruption Response Dashboard

**What:** Frontend page showing active disruptions, recommendations, and response actions with approval workflow.

**Testing:**
- Active disruptions listed with severity badges and affected product count
- Expanding a disruption shows recommendations with action buttons (Approve/Reject)
- Approving a re-source recommendation creates a draft PO with the alternative supplier
- Timeline view shows disruption detection -> recommendations -> approvals -> resolution
- Email/webhook notification sent when critical disruption detected

### Definition of Done -- Phase 7
- [ ] LLM parses supplier news and ESG reports into structured risk signals
- [ ] Disruption detector identifies risk escalations, PO delays, and stockout projections
- [ ] AI-generated recommendations include re-source, expedite, and safety stock options
- [ ] Recommendations include cost impact estimates
- [ ] Planner approval workflow: pending -> approved -> executed
- [ ] Disruption dashboard shows active incidents with timeline view
- [ ] Notification system alerts planners of critical disruptions
- [ ] Test suite: disruption detection tested with simulated scenarios

---

## Phase 8: Enterprise Integration & Compliance

**Duration:** 3-4 weeks
**Dependencies:** Phase 5 (REST API), Phase 7 (risk scoring)
**Goal:** EDI X12 ingest, OIDC SSO, CSDDD/LkSG compliance reporting, and webhook notifications.

### Task 8.1: EDI X12 Parser

**What:** Parse EDI X12 850 (Purchase Order), 855 (PO Acknowledgement), and 856 (ASN) transaction sets into platform data.

**Design:**
```python
# backend/app/integrations/edi_parser.py
class X12Parser:
    def parse_850(self, edi_content: str) -> PurchaseOrderCreate:
        """Parse X12 850 Purchase Order into platform PO schema."""
        segments = self._tokenize(edi_content)
        po = PurchaseOrderCreate(
            po_number=self._get_segment_value(segments, "BEG", 3),
            order_date=self._parse_date(self._get_segment_value(segments, "BEG", 5)),
            lines=self._parse_po_lines(segments),
            logistics_data={"edi_type": "x12_850", "raw_isa": segments[0]},
        )
        return po

    def parse_856(self, edi_content: str) -> ShipmentNotification:
        """Parse X12 856 Advance Ship Notice."""
        ...
```

**Testing:**
- Valid X12 850 sample parsed into correct PO number, date, and line items
- Invalid EDI (missing required segments) returns descriptive parsing error
- SSCC extracted from 856 ASN and stored in `logistics_data`
- Multiple line items (N1, PO1 loops) parsed correctly
- Date formats (CCYYMMDD) parsed to ISO 8601

### Task 8.2: OpenID Connect SSO

**What:** Support enterprise SSO via OpenID Connect for customer identity providers (Azure AD, Okta, Google Workspace).

**Design:**
```python
# backend/app/api/v1/auth.py
@router.get("/auth/sso/{tenant_slug}")
async def initiate_sso(tenant_slug: str, ...):
    """Redirect to tenant's configured OIDC provider."""
    tenant = await get_tenant_by_slug(tenant_slug)
    oidc_config = tenant.settings.get("oidc", {})
    authorization_url = build_oidc_auth_url(
        issuer=oidc_config["issuer"],
        client_id=oidc_config["client_id"],
        redirect_uri=f"{BASE_URL}/auth/callback",
        scope="openid email profile",
    )
    return RedirectResponse(authorization_url)

@router.get("/auth/callback")
async def sso_callback(code: str, state: str, ...):
    """Exchange authorization code for tokens, create/update user, return JWT."""
    ...
```

**Testing:**
- SSO initiation redirects to configured OIDC provider with correct parameters
- Callback exchanges code for tokens and creates user if not exists
- Existing user login via SSO updates last_login_at without creating duplicate
- Tenant without OIDC configuration returns 404 on SSO initiation
- CSRF protection via state parameter validated on callback
- JWT issued after SSO contains same claims as password-based login

### Task 8.3: CSDDD/LkSG Compliance Reporting

**What:** Generate regulatory compliance reports showing supplier due diligence status, risk assessments, and corrective actions for CSDDD/LkSG requirements.

**Design:**
```python
# backend/app/api/v1/compliance.py
@router.get("/compliance/csddd/report")
async def generate_csddd_report(
    reporting_year: int = Query(...),
    format: str = Query("json", regex="^(json|csv|pdf)$"),
    ...
):
    """
    Generate CSDDD annual due diligence report covering:
    - Tier 1 and Tier 2 supplier risk assessments
    - Human rights and environmental risk findings
    - Corrective actions taken
    - Geographic risk distribution
    """
    suppliers = await get_all_active_suppliers(tenant_id)
    risk_assessments = await get_risk_assessments(tenant_id, reporting_year)
    report = CSDDDReport(
        reporting_year=reporting_year,
        total_suppliers=len(suppliers),
        assessed_suppliers=len([s for s in suppliers if s.compliance_data.get("csddd")]),
        high_risk_suppliers=len([s for s in suppliers if s.risk_tier in ("high", "critical")]),
        corrective_actions=await get_corrective_actions(tenant_id, reporting_year),
        geographic_distribution=compute_geographic_distribution(suppliers),
    )
    return report
```

**Testing:**
- Report includes all active suppliers with their risk tiers
- Unassessed suppliers flagged as compliance gaps
- Geographic distribution shows supplier count by country
- CSV export contains all required CSDDD data fields
- Report covers correct reporting year (filters by assessment date)

### Task 8.4: Webhook & Notification System

**What:** Configurable webhooks and email notifications for disruptions, stockout alerts, and forecast generation completions.

**Testing:**
- Webhook fires on disruption detection with correct JSON payload
- Failed webhook delivery retried up to 3 times with exponential backoff
- Email notification sent for critical disruptions to configured recipients
- Notification preferences stored per user in `app_user.preferences` JSONB
- Webhook signature (HMAC-SHA256) included for payload verification

### Definition of Done -- Phase 8
- [ ] EDI X12 850 and 856 parsed into platform data structures
- [ ] OIDC SSO works with Azure AD test tenant
- [ ] CSDDD compliance report generates with all required fields
- [ ] Webhooks fire on configurable events with retry logic
- [ ] Email notifications for critical alerts
- [ ] Test suite: EDI parser tested against real-world X12 samples

---

## Phase 9: S&OP Intelligence

**Duration:** 3-4 weeks
**Dependencies:** Phase 4 (advanced ML), Phase 3 (scenario modeling)
**Goal:** LLM-generated S&OP narrative commentary, natural language query interface for ad-hoc analysis, and executive dashboards.

### Task 9.1: S&OP Narrative Generation

**What:** Auto-generate variance explanations comparing current period actuals to prior forecasts, highlighting key drivers and exceptions.

**Design:**
```python
# backend/app/services/sop_service.py
class SOPNarrativeGenerator:
    async def generate_commentary(self, tenant_id: UUID, period: DateRange) -> str:
        """
        Generate S&OP narrative from planning data:
        1. Compare actuals vs forecast for the period
        2. Identify top variance drivers (products, regions)
        3. Summarize inventory health and supplier performance
        4. Generate natural language commentary via LLM
        """
        variance_data = await self.compute_variance(tenant_id, period)
        kpis = await self.get_kpi_snapshot(tenant_id, period)
        
        prompt = f"""You are a supply chain analyst. Generate a concise S&OP executive 
        summary for the period {period.start} to {period.end}. 
        
        Key metrics:
        - Overall forecast accuracy (WMAPE): {kpis.wmape:.1%}
        - Fill rate: {kpis.fill_rate:.1%}
        - Inventory turns: {kpis.inventory_turns:.1f}
        
        Top positive variances (demand exceeded forecast):
        {self._format_variances(variance_data.positive_top5)}
        
        Top negative variances (demand below forecast):
        {self._format_variances(variance_data.negative_top5)}
        
        Supplier issues: {self._format_supplier_issues(variance_data.supplier_issues)}
        
        Write 3-4 paragraphs covering: demand performance, inventory health, 
        supply risks, and recommended actions. Use SCOR metric terminology."""
        
        narrative = await self.llm.generate(prompt)
        return narrative
```

**Testing:**
- Narrative references correct WMAPE and fill rate values from KPI data
- Top variance products mentioned by name
- Narrative uses SCOR terminology (perfect order fulfilment, fill rate, inventory turns)
- Narrative length between 200-500 words
- Narrative stored for retrieval and inclusion in S&OP dashboards

### Task 9.2: Natural Language Query Interface

**What:** Allow planners to ask questions in natural language (e.g., "What is the forecast accuracy for electronics products in Q2?") and receive data-backed answers.

**Design:**
```python
# backend/app/api/v1/query.py
@router.post("/query", response_model=QueryResponse)
async def natural_language_query(query: NLQueryRequest, ...):
    """
    Convert natural language question to SQL, execute, format response.
    Uses LLM to generate SQL from schema context + question.
    """
    sql = await self.text_to_sql(query.question, schema_context)
    # Validate SQL is read-only (SELECT only, no mutations)
    if not is_read_only_sql(sql):
        raise HTTPException(400, "Only read queries are supported")
    results = await db.execute(text(sql))
    formatted = await self.format_response(query.question, results, sql)
    return QueryResponse(answer=formatted, sql=sql, data=results)
```

**Testing:**
- "What is the MAPE for product X?" returns correct MAPE value
- "Show me top 5 products by forecast error" returns ranked list
- Mutation queries (INSERT, UPDATE, DELETE) are rejected
- SQL injection attempts blocked by read-only validation
- Question about non-existent product returns "No data found" message

### Definition of Done -- Phase 9
- [ ] S&OP narrative auto-generated from planning data with SCOR terminology
- [ ] Natural language query converts questions to SQL with read-only enforcement
- [ ] Executive dashboard with period-over-period comparison charts
- [ ] Narrative and query results accessible via API and frontend
- [ ] Test suite: narrative quality evaluated against expected content coverage

---

## Phase 10: Multi-Enterprise & Sustainability

**Duration:** 4-5 weeks
**Dependencies:** Phase 8 (enterprise integration), Phase 6 (graph layer)
**Goal:** Trading partner data sharing API, sustainability/emissions tracking, EPCIS 2.0 integration, and managed cloud deployment.

### Task 10.1: Multi-Enterprise Data Sharing API

**What:** API enabling trading partners to share demand signals, inventory visibility, and shipment status with configured permissions.

**Design:**
```python
# backend/app/api/v1/collaboration.py
@router.post("/partners/{partner_id}/share")
async def share_data_with_partner(
    partner_id: UUID,
    share_config: DataShareConfig,  # {entity_types: ["forecast", "inventory"], scope: {...}}
    ...
):
    """
    Share specified data with trading partner via authenticated API.
    Partner permissions define what data categories they can access.
    """
    ...

@router.get("/partners/{partner_id}/shared-data")
async def get_shared_data(partner_id: UUID, entity_type: str, ...):
    """Retrieve data shared by a partner (their forecasts/inventory visible to us)."""
    ...
```

**Testing:**
- Partner A shares forecast data with Partner B; B can read A's forecasts for shared products
- Unshared products/locations are not visible to partner
- Partner authentication uses separate API keys with scoped permissions
- Data sharing is bidirectional and independently configurable

### Task 10.2: Sustainability & Emissions Tracking

**What:** Track Scope 3 supply chain emissions alongside financial metrics. Compute carbon intensity per product using supplier-reported emissions data.

**Design:**
```python
# Sustainability metrics stored in kpi_snapshot.extended_metrics JSONB:
# {"sustainability": {
#     "scope3_emissions_kg": 12500,
#     "carbon_intensity_per_unit": 0.25,
#     "renewable_energy_pct": 0.65,
#     "waste_diversion_rate": 0.82
# }}

# Supplier emissions stored in supplier.compliance_data JSONB:
# {"emissions": {
#     "scope1_co2_tonnes": 500,
#     "scope2_co2_tonnes": 200,
#     "report_year": 2025,
#     "methodology": "GHG Protocol"
# }}
```

**Testing:**
- Scope 3 emissions calculated from supplier-reported data and purchase volumes
- Carbon intensity per product computed as total emissions / units produced
- Sustainability KPIs displayed alongside financial KPIs on dashboards
- Missing emissions data for a supplier flagged as data gap

### Task 10.3: EPCIS 2.0 Event Export

**What:** Export supply chain events in GS1 EPCIS 2.0 JSON format for trading partner traceability and FDA FSMA compliance.

**Testing:**
- Inventory receipt event exports as valid EPCIS ObjectEvent with What/When/Where/Why
- SSCC and GTIN identifiers correctly formatted in EPCIS output
- EPCIS JSON validates against official GS1 EPCIS 2.0 JSON-LD schema

### Task 10.4: Managed Cloud Deployment

**What:** Production-ready cloud deployment with Kubernetes manifests, Terraform infrastructure, monitoring, and CI/CD pipeline.

**Testing:**
- Kubernetes deployment starts all services (API, worker, frontend, PostgreSQL)
- Health check endpoints return 200 for all services
- Auto-scaling triggers at 70% CPU utilization
- Database backup runs daily with point-in-time recovery tested
- CI/CD pipeline: push to main triggers build, test, deploy to staging

### Definition of Done -- Phase 10
- [ ] Trading partner API with scoped data sharing and authentication
- [ ] Sustainability metrics tracked and displayed alongside financial KPIs
- [ ] EPCIS 2.0 event export validates against GS1 schema
- [ ] Kubernetes deployment with auto-scaling and monitoring
- [ ] CI/CD pipeline with automated testing and staging deployment
- [ ] Load testing: API handles 500 concurrent users with p95 latency < 500ms
- [ ] Security audit: OWASP API Top 10 addressed
- [ ] Documentation: API reference, deployment guide, user guide published

---

## Summary

| Phase | Duration | Key Deliverables |
|-------|----------|-----------------|
| 1. Foundation | 4-5 weeks | Multi-tenant API, master data CRUD, CSV import, auth, frontend shell |
| 2. Demand Forecasting | 4-5 weeks | LightGBM forecasting, accuracy metrics, override workflow, forecast UI |
| 3. Inventory & Scenarios | 3-4 weeks | Safety stock, reorder points, what-if scenarios, replenishment recs |
| 4. Advanced ML | 4-5 weeks | TFT model, SHAP explainability, external signals, ensembling |
| 5. Supplier Performance | 3-4 weeks | OTIF metrics, structured risk scoring, supplier dashboards, REST API |
| 6. Graph Layer | 3-4 weeks | Network graph, multi-tier tracing, impact analysis, visualization |
| 7. Agentic AI | 4-5 weeks | LLM risk scoring, disruption detection, AI response recommendations |
| 8. Enterprise Integration | 3-4 weeks | EDI X12, OIDC SSO, CSDDD compliance, webhooks |
| 9. S&OP Intelligence | 3-4 weeks | Narrative generation, NL query, executive dashboards |
| 10. Multi-Enterprise | 4-5 weeks | Partner data sharing, sustainability tracking, EPCIS, cloud deployment |

**Total estimated duration:** 36-45 weeks (9-11 months)

**MVP (Phases 1-3):** 11-14 weeks (3-3.5 months) -- delivers demand forecasting, inventory planning, and scenario modeling.

**Competitive differentiation (Phases 4-7):** Additional 14-18 weeks (3.5-4.5 months) -- delivers the AI-native capabilities (TFT forecasting, SHAP explainability, agentic disruption response) that distinguish this platform from frePPLe and position it below Blue Yonder/Kinaxis on price.
