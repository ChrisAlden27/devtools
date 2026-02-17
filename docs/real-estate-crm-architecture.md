# Real Estate CRM with Sales Analytics AI — Solution Blueprint

## 1) Full System Architecture

### High-level architecture

```text
[React Mobile-First SPA]
        |
        | HTTPS REST + JWT
        v
[Spring Boot API Gateway / Core Backend]
   |            |             |
   |            |             +--> [Redis (optional cache/session/rate-limit)]
   |            |
   |            +--> [PostgreSQL]
   |
   +--> async events (REST or queue)
               |
               v
      [FastAPI AI Microservice]
               |
               +--> [ML Models + Feature Store tables in PostgreSQL]

Observability: Prometheus + Grafana + Loki (or ELK)
Deployment: Docker Compose on Singapore VPS + Nginx reverse proxy + SSL
```

### Component responsibilities

- **React Frontend (mobile-first):**
  - Lead list/detail, Kanban pipeline, inventory, commissions, dashboards.
  - Optimized for low-bandwidth connections and smaller screens.
  - Uses token-based auth (access + refresh).
- **Spring Boot Backend:**
  - Domain-driven modules: leads, pipeline, properties, commissions, analytics.
  - Exposes REST APIs.
  - Manages business rules (commission logic, stage transitions, permissions).
  - Produces AI feature snapshots and consumes prediction results.
- **PostgreSQL:**
  - Primary system of record.
  - Normalized transactional tables + denormalized reporting views/materialized views.
- **FastAPI AI Service:**
  - Endpoints for close-probability scoring and revenue forecasting.
  - Batch retraining job + online inference.
  - Writes predictions back to `ai_predictions`.
- **Infrastructure:**
  - Dockerized services.
  - Nginx for TLS termination and routing (`/api`, `/ai`, `/`).
  - Backups and log rotation.

### Security and production readiness

- JWT auth + role-based access (`ADMIN`, `MANAGER`, `AGENT`).
- BCrypt password hashing.
- Row-level ownership constraints (agents see own leads unless elevated role).
- API rate limiting (Nginx or Spring filter).
- Daily DB backup + point-in-time recovery strategy.
- Health checks: `/actuator/health` (backend), `/health` (AI).

---

## 2) Database Schema (SQL)

```sql
-- Enable UUID extension
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

-- =========================
-- Core identity / tenancy
-- =========================
CREATE TABLE agencies (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name VARCHAR(120) NOT NULL,
    country_code VARCHAR(5) DEFAULT 'MM',
    city VARCHAR(80),
    timezone VARCHAR(50) DEFAULT 'Asia/Yangon',
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    agency_id UUID NOT NULL REFERENCES agencies(id) ON DELETE CASCADE,
    full_name VARCHAR(120) NOT NULL,
    email VARCHAR(120) UNIQUE NOT NULL,
    phone VARCHAR(30),
    password_hash VARCHAR(255) NOT NULL,
    role VARCHAR(20) NOT NULL CHECK (role IN ('ADMIN','MANAGER','AGENT')),
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW()
);

-- =========================
-- Leads and pipeline
-- =========================
CREATE TABLE leads (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    agency_id UUID NOT NULL REFERENCES agencies(id) ON DELETE CASCADE,
    assigned_agent_id UUID REFERENCES users(id) ON DELETE SET NULL,
    source VARCHAR(50) NOT NULL, -- facebook, website, referral, walk-in
    full_name VARCHAR(120) NOT NULL,
    phone VARCHAR(30),
    email VARCHAR(120),
    budget_min NUMERIC(14,2),
    budget_max NUMERIC(14,2),
    preferred_township VARCHAR(100),
    intent VARCHAR(20) CHECK (intent IN ('BUY','RENT','SELL')),
    status VARCHAR(20) NOT NULL DEFAULT 'NEW' CHECK (
      status IN ('NEW','CONTACTED','QUALIFIED','NEGOTIATION','WON','LOST')
    ),
    lost_reason VARCHAR(255),
    notes TEXT,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE TABLE pipeline_stages (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    agency_id UUID NOT NULL REFERENCES agencies(id) ON DELETE CASCADE,
    name VARCHAR(80) NOT NULL,
    stage_order INT NOT NULL,
    probability_default NUMERIC(5,2) CHECK (probability_default >= 0 AND probability_default <= 100),
    is_closed_stage BOOLEAN NOT NULL DEFAULT FALSE,
    UNIQUE (agency_id, name),
    UNIQUE (agency_id, stage_order)
);

CREATE TABLE opportunities (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    agency_id UUID NOT NULL REFERENCES agencies(id) ON DELETE CASCADE,
    lead_id UUID NOT NULL REFERENCES leads(id) ON DELETE CASCADE,
    property_id UUID,
    owner_agent_id UUID REFERENCES users(id),
    stage_id UUID NOT NULL REFERENCES pipeline_stages(id),
    expected_value NUMERIC(14,2) NOT NULL,
    expected_close_date DATE,
    close_probability NUMERIC(5,2) CHECK (close_probability >= 0 AND close_probability <= 100),
    status VARCHAR(20) NOT NULL DEFAULT 'OPEN' CHECK (status IN ('OPEN','WON','LOST')),
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE TABLE opportunity_stage_history (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    opportunity_id UUID NOT NULL REFERENCES opportunities(id) ON DELETE CASCADE,
    from_stage_id UUID REFERENCES pipeline_stages(id),
    to_stage_id UUID REFERENCES pipeline_stages(id),
    changed_by UUID REFERENCES users(id),
    changed_at TIMESTAMP NOT NULL DEFAULT NOW(),
    note TEXT
);

-- =========================
-- Properties / inventory
-- =========================
CREATE TABLE properties (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    agency_id UUID NOT NULL REFERENCES agencies(id) ON DELETE CASCADE,
    listing_code VARCHAR(40) NOT NULL,
    title VARCHAR(160) NOT NULL,
    property_type VARCHAR(30) NOT NULL, -- condo, land, house, office
    township VARCHAR(100) NOT NULL,
    address TEXT,
    bedrooms INT,
    bathrooms INT,
    floor_area_sqft NUMERIC(10,2),
    lot_area_sqft NUMERIC(10,2),
    price NUMERIC(14,2) NOT NULL,
    listing_type VARCHAR(20) NOT NULL CHECK (listing_type IN ('SALE','RENT')),
    status VARCHAR(20) NOT NULL DEFAULT 'ACTIVE' CHECK (status IN ('ACTIVE','PENDING','SOLD','RENTED','OFF_MARKET')),
    owner_name VARCHAR(120),
    owner_phone VARCHAR(30),
    created_by UUID REFERENCES users(id),
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW(),
    UNIQUE (agency_id, listing_code)
);

CREATE TABLE property_images (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    property_id UUID NOT NULL REFERENCES properties(id) ON DELETE CASCADE,
    image_url TEXT NOT NULL,
    sort_order INT NOT NULL DEFAULT 0,
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
);

-- =========================
-- Deals and commissions
-- =========================
CREATE TABLE deals (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    agency_id UUID NOT NULL REFERENCES agencies(id) ON DELETE CASCADE,
    opportunity_id UUID NOT NULL UNIQUE REFERENCES opportunities(id) ON DELETE CASCADE,
    property_id UUID REFERENCES properties(id),
    won_by_agent_id UUID NOT NULL REFERENCES users(id),
    deal_value NUMERIC(14,2) NOT NULL,
    commission_rate NUMERIC(6,3) NOT NULL, -- e.g. 2.500%
    gross_commission NUMERIC(14,2) NOT NULL,
    net_commission NUMERIC(14,2),
    closed_at TIMESTAMP NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE TABLE commission_rules (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    agency_id UUID NOT NULL REFERENCES agencies(id) ON DELETE CASCADE,
    name VARCHAR(80) NOT NULL,
    min_amount NUMERIC(14,2),
    max_amount NUMERIC(14,2),
    commission_rate NUMERIC(6,3) NOT NULL,
    agent_share_rate NUMERIC(6,3) NOT NULL,
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE TABLE commission_payouts (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    agency_id UUID NOT NULL REFERENCES agencies(id) ON DELETE CASCADE,
    deal_id UUID NOT NULL REFERENCES deals(id) ON DELETE CASCADE,
    agent_id UUID NOT NULL REFERENCES users(id),
    payout_amount NUMERIC(14,2) NOT NULL,
    payout_status VARCHAR(20) NOT NULL DEFAULT 'PENDING' CHECK (payout_status IN ('PENDING','APPROVED','PAID','REJECTED')),
    approved_by UUID REFERENCES users(id),
    approved_at TIMESTAMP,
    paid_at TIMESTAMP,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW()
);

-- =========================
-- Activity + AI
-- =========================
CREATE TABLE activities (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    agency_id UUID NOT NULL REFERENCES agencies(id) ON DELETE CASCADE,
    lead_id UUID REFERENCES leads(id) ON DELETE CASCADE,
    opportunity_id UUID REFERENCES opportunities(id) ON DELETE CASCADE,
    agent_id UUID REFERENCES users(id),
    activity_type VARCHAR(30) NOT NULL, -- CALL, VISIT, FOLLOW_UP, MESSAGE
    subject VARCHAR(120),
    due_at TIMESTAMP,
    completed_at TIMESTAMP,
    status VARCHAR(20) NOT NULL DEFAULT 'OPEN' CHECK (status IN ('OPEN','DONE','CANCELLED')),
    notes TEXT,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE TABLE ai_predictions (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    agency_id UUID NOT NULL REFERENCES agencies(id) ON DELETE CASCADE,
    opportunity_id UUID REFERENCES opportunities(id) ON DELETE CASCADE,
    prediction_type VARCHAR(40) NOT NULL, -- CLOSE_PROBABILITY, REVENUE_FORECAST
    model_version VARCHAR(40) NOT NULL,
    prediction_value NUMERIC(14,4) NOT NULL,
    confidence_score NUMERIC(5,2),
    input_snapshot JSONB NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
);

-- =========================
-- Helpful indexes
-- =========================
CREATE INDEX idx_leads_agency_status ON leads(agency_id, status);
CREATE INDEX idx_leads_assigned_agent ON leads(assigned_agent_id);
CREATE INDEX idx_opportunities_agency_stage ON opportunities(agency_id, stage_id);
CREATE INDEX idx_opportunities_owner_agent ON opportunities(owner_agent_id);
CREATE INDEX idx_properties_agency_status ON properties(agency_id, status);
CREATE INDEX idx_deals_closed_at ON deals(closed_at);
CREATE INDEX idx_ai_predictions_oppty_type ON ai_predictions(opportunity_id, prediction_type);
```

---

## 3) Backend Project Structure (Spring Boot)

```text
backend/
  src/main/java/com/agency/crm/
    config/
      SecurityConfig.java
      OpenApiConfig.java
      JacksonConfig.java
    common/
      exception/
      response/
      util/
    auth/
      controller/
      service/
      dto/
      entity/
      repository/
    user/
      controller/
      service/
      dto/
      entity/
      repository/
    lead/
      controller/
      service/
      dto/
      entity/
      repository/
      mapper/
    pipeline/
      controller/
      service/
      dto/
      entity/
      repository/
    property/
      controller/
      service/
      dto/
      entity/
      repository/
    commission/
      controller/
      service/
      dto/
      entity/
      repository/
    analytics/
      controller/
      service/
      dto/
      repository/
    ai/
      client/
      service/
      dto/
    audit/
      entity/
      repository/
      listener/
    CrmApplication.java
  src/main/resources/
    application.yml
    application-prod.yml
    db/migration/ (Flyway SQL files)
  Dockerfile
  pom.xml
```

**Architecture style:** Hexagonal-lite / layered modular monolith for speed and maintainability.
**Why:** small team can ship fast, later split into microservices if load grows.

---

## 4) API Endpoint List (REST)

### Auth

- `POST /api/v1/auth/login`
- `POST /api/v1/auth/refresh`
- `POST /api/v1/auth/logout`
- `GET /api/v1/auth/me`

### Users / Agents

- `GET /api/v1/users`
- `POST /api/v1/users`
- `GET /api/v1/users/{id}`
- `PUT /api/v1/users/{id}`
- `PATCH /api/v1/users/{id}/status`
- `GET /api/v1/agents/performance?from=&to=`

### Leads

- `GET /api/v1/leads?status=&source=&agentId=&page=`
- `POST /api/v1/leads`
- `GET /api/v1/leads/{id}`
- `PUT /api/v1/leads/{id}`
- `PATCH /api/v1/leads/{id}/assign`
- `PATCH /api/v1/leads/{id}/status`
- `POST /api/v1/leads/{id}/activities`

### Pipeline / Opportunities (Kanban)

- `GET /api/v1/pipeline/stages`
- `POST /api/v1/pipeline/stages`
- `PATCH /api/v1/pipeline/stages/reorder`
- `GET /api/v1/opportunities?stageId=&agentId=`
- `POST /api/v1/opportunities`
- `GET /api/v1/opportunities/{id}`
- `PUT /api/v1/opportunities/{id}`
- `PATCH /api/v1/opportunities/{id}/move-stage`
- `PATCH /api/v1/opportunities/{id}/mark-won`
- `PATCH /api/v1/opportunities/{id}/mark-lost`

### Properties / Inventory

- `GET /api/v1/properties?status=&township=&type=&listingType=`
- `POST /api/v1/properties`
- `GET /api/v1/properties/{id}`
- `PUT /api/v1/properties/{id}`
- `PATCH /api/v1/properties/{id}/status`
- `POST /api/v1/properties/{id}/images`
- `DELETE /api/v1/properties/{id}/images/{imageId}`

### Deals / Commissions

- `GET /api/v1/deals?from=&to=&agentId=`
- `POST /api/v1/deals`
- `GET /api/v1/deals/{id}`
- `GET /api/v1/commissions/rules`
- `POST /api/v1/commissions/rules`
- `PUT /api/v1/commissions/rules/{id}`
- `POST /api/v1/commissions/calculate`
- `GET /api/v1/commissions/payouts?status=`
- `PATCH /api/v1/commissions/payouts/{id}/approve`
- `PATCH /api/v1/commissions/payouts/{id}/pay`

### Analytics Dashboard

- `GET /api/v1/analytics/overview?from=&to=`
- `GET /api/v1/analytics/sales-trend?interval=day|week|month`
- `GET /api/v1/analytics/conversion-funnel?from=&to=`
- `GET /api/v1/analytics/agent-leaderboard?from=&to=`
- `GET /api/v1/analytics/source-performance?from=&to=`
- `GET /api/v1/analytics/revenue-forecast?months=3`

### AI Insights Integration

- `POST /api/v1/ai/opportunities/{id}/score-close-probability`
- `POST /api/v1/ai/forecast/revenue`
- `GET /api/v1/ai/predictions/opportunities/{id}`

---

## 5) Frontend Folder Structure (React, mobile-first)

```text
frontend/
  public/
  src/
    app/
      store.ts
      router.tsx
      providers.tsx
    assets/
    components/
      ui/               # buttons, inputs, cards, sheets, badges
      layout/           # AppShell, MobileBottomNav, Header
      charts/
    features/
      auth/
        api.ts
        hooks.ts
        pages/
      leads/
        api.ts
        components/
        pages/
      pipeline/
        api.ts
        components/
        pages/
      properties/
        api.ts
        components/
        pages/
      commissions/
        api.ts
        components/
        pages/
      analytics/
        api.ts
        components/
        pages/
      ai-insights/
        api.ts
        components/
    hooks/
    lib/
      http.ts           # axios instance, interceptors
      format.ts
      constants.ts
    styles/
      globals.css
      tokens.css
    types/
    main.tsx
  Dockerfile
  package.json
```

Mobile-first guidelines:

- Base breakpoint starts at 360px width.
- Use bottom navigation for key modules.
- Keep forms single-column.
- Use expandable cards instead of wide tables.
- Show KPI cards in 2-column grid max on small devices.

---

## 6) AI Microservice Structure (FastAPI)

```text
ai-service/
  app/
    main.py
    api/
      v1/
        endpoints/
          health.py
          predict.py
          forecast.py
    core/
      config.py
      logging.py
      security.py
    schemas/
      predict.py
      forecast.py
      common.py
    services/
      feature_builder.py
      predictor.py
      forecaster.py
      model_registry.py
    models/
      close_probability_model.pkl
      revenue_forecast_model.pkl
    db/
      session.py
      repository.py
    tasks/
      retrain.py
  tests/
    test_predict.py
    test_forecast.py
  requirements.txt
  Dockerfile
```

### FastAPI endpoints

- `GET /health`
- `POST /v1/predict/close-probability`
- `POST /v1/predict/revenue-forecast`
- `POST /v1/train/retrain` (protected/admin only)

### AI flow

1. Backend sends structured opportunity + activity features.
2. FastAPI validates payload with Pydantic.
3. Model inference returns probability / forecast + confidence.
4. Backend persists to `ai_predictions` and surfaces on dashboard.

---

## 7) Step-by-Step Build Plan

### Phase 0 — Foundation (Week 1)

1. Set up mono-repo folders: `backend`, `frontend`, `ai-service`, `infra`.
2. Configure Docker Compose (Postgres, backend, frontend, ai, nginx).
3. Add CI (lint + test + build images).
4. Define coding standards, branching strategy, `.env` templates.

### Phase 1 — Core CRM (Weeks 2–4)

1. Implement auth + RBAC.
2. Build lead CRUD + assignment + activity logging.
3. Build pipeline stages + drag/drop Kanban transitions.
4. Build property inventory CRUD + media upload.
5. Add Flyway migrations and seed data.

### Phase 2 — Revenue Operations (Weeks 5–6)

1. Deal creation from won opportunity.
2. Commission rules engine + payout workflow.
3. Agent performance module (KPIs: won deals, conversion, revenue).

### Phase 3 — Analytics + AI (Weeks 7–8)

1. Build analytics aggregates + dashboard APIs.
2. Implement FastAPI scoring endpoints.
3. Integrate backend ↔ AI service with retry/timeouts/circuit breaker.
4. Show close probability + forecast cards in UI.

### Phase 4 — Production Hardening (Week 9)

1. Security hardening: CORS, rate limit, audit logs.
2. Load/performance testing (Locust/JMeter + k6 for API).
3. Backup/restore drills and monitoring alerts.
4. UAT with pilot agency; collect feedback.

### Phase 5 — Go-live and Iteration (Week 10+)

1. Deploy to Singapore VPS with blue/green or rolling strategy.
2. Enable error tracking and usage analytics.
3. Plan v2 features (WhatsApp integration, Myanmar language UI, automated reminders).

---

## Recommended Docker Compose Services

- `nginx`
- `frontend`
- `backend`
- `ai-service`
- `postgres`
- `redis` (optional but recommended)
- `prometheus` + `grafana` (optional, production recommended)

This design gives a practical, production-ready path for small Myanmar real estate agencies while keeping complexity manageable.
