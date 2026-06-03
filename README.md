# FinSight — Production Architecture Blueprint

This repository contains the target architecture for a **financial statement analysis platform** focused on publicly traded companies.

## 1) Architecture Style

- **Domain Driven Design (DDD)** with bounded contexts:
  - Ingestion
  - Document Intelligence
  - Financial Modeling
  - AI Research
  - Identity & Access
- **Clean Architecture** per service:
  - `domain` (entities, value objects, domain services)
  - `application` (use cases, orchestration)
  - `infrastructure` (SQLAlchemy repos, Redis cache, RabbitMQ adapters, OpenAI clients)
  - `interfaces` (FastAPI REST, background message handlers)
- **Event-driven + async processing** using RabbitMQ + Celery for long-running work.

---

## 2) C4 — System Context Diagram

```mermaid
flowchart LR
    Investor[Investor / Analyst]
    Admin[Platform Admin]
    FinSight[FinSight Platform]
    EDGAR[SEC EDGAR / Company IR Sites]
    OpenAI[OpenAI API]
    Email[Email / Notification Provider]

    Investor -->|Upload reports, request analysis, view dashboards| FinSight
    Admin -->|Manage tenants, users, limits, prompts| FinSight
    FinSight -->|Fetch source filings metadata (optional)| EDGAR
    FinSight -->|LLM inference for qualitative analysis| OpenAI
    FinSight -->|Send completion/failure notifications| Email
```

---

## 3) C4 — Container Diagram

```mermaid
flowchart TB
    subgraph Client
      Web[Next.js + React + TypeScript]
    end

    subgraph Platform
      API[FastAPI Gateway / BFF\nPython 3.13]
      Auth[Auth Service\nJWT + RBAC]
      Ingestion[Ingestion Service\nUpload, storage metadata]
      DocProc[Document Processing Service\nPyMuPDF/pdfplumber/OCR]
      Metrics[Financial Metrics Service\nRatios, scorecards]
      AI[AI Analysis Service\nPydanticAI + OpenAI]
      Report[Report Service\nAssembled outputs]
      Worker[Celery Workers]
      Beat[Celery Beat]
    end

    subgraph Data
      PG[(PostgreSQL)]
      Redis[(Redis)]
      RMQ[(RabbitMQ)]
      S3[(AWS S3)]
    end

    Web --> API
    API --> Auth
    API --> Ingestion
    API --> Metrics
    API --> Report

    Ingestion --> S3
    Ingestion --> PG
    Ingestion --> RMQ

    RMQ --> Worker
    Beat --> Worker

    Worker --> DocProc
    Worker --> Metrics
    Worker --> AI

    DocProc --> PG
    DocProc --> Redis
    Metrics --> PG
    Metrics --> Redis
    AI --> OpenAIAPI[OpenAI API]
    AI --> PG

    Report --> PG
    Report --> Redis
```

---

## 4) C4 — Component Diagram (Document Processing Service)

```mermaid
flowchart LR
    API[FastAPI Endpoint\nPOST /v1/reports]
    Cmd[UploadReportCommand Handler]
    Repo[(SQLAlchemy Repositories)]
    Obj[S3 Object Store Adapter]
    Bus[RabbitMQ Publisher]

    PDF[PDF Extractor\nPyMuPDF + pdfplumber]
    OCR[OCR Adapter\nTesseract/Textract]
    Norm[Statement Normalizer]
    Val[Validation Engine\nPydantic v2 schemas]

    API --> Cmd
    Cmd --> Repo
    Cmd --> Obj
    Cmd --> Bus

    Bus --> PDF
    PDF --> OCR
    OCR --> Norm
    Norm --> Val
    Val --> Repo
```

---

## 5) Sequence Diagrams

### 5.1 Report Upload

```mermaid
sequenceDiagram
    autonumber
    participant U as User
    participant W as Next.js Web
    participant A as FastAPI
    participant S3 as S3
    participant DB as PostgreSQL
    participant MQ as RabbitMQ

    U->>W: Upload PDF + company + period
    W->>A: POST /v1/reports (metadata)
    A->>S3: Store raw PDF
    A->>DB: Insert report(status=UPLOADED)
    A->>MQ: Publish report.uploaded
    A-->>W: 202 Accepted + report_id
    W-->>U: Show processing state
```

### 5.2 Document Processing

```mermaid
sequenceDiagram
    autonumber
    participant C as Celery Worker
    participant MQ as RabbitMQ
    participant S3 as S3
    participant X as Extractor(PyMuPDF/pdfplumber)
    participant O as OCR
    participant DB as PostgreSQL

    MQ-->>C: report.uploaded
    C->>S3: Download PDF
    C->>X: Parse text/tables
    alt text quality low
      C->>O: OCR fallback
    end
    C->>DB: Save statements + line_items
    C->>MQ: Publish report.processed
```

### 5.3 Ratio Calculation

```mermaid
sequenceDiagram
    autonumber
    participant C as Celery Worker
    participant MQ as RabbitMQ
    participant DB as PostgreSQL
    participant R as Ratio Engine

    MQ-->>C: report.processed
    C->>DB: Load normalized financials
    C->>R: Compute ratios/metrics
    R-->>C: ratio set + investment scores
    C->>DB: Persist metrics snapshot
    C->>MQ: Publish metrics.calculated
```

### 5.4 AI Report Generation

```mermaid
sequenceDiagram
    autonumber
    participant C as Celery Worker
    participant MQ as RabbitMQ
    participant DB as PostgreSQL
    participant P as Prompt Builder (PydanticAI)
    participant O as OpenAI API

    MQ-->>C: metrics.calculated
    C->>DB: Load financial history + ratios
    C->>P: Build structured prompt
    P->>O: Request analysis
    O-->>P: Structured response
    C->>DB: Save investment report
    C->>MQ: Publish analysis.completed
```

---

## 6) Database Schema (PostgreSQL, SQLAlchemy 2, Alembic)

### Core Tables

- `tenants(id, name, plan, created_at)`
- `users(id, tenant_id, email, password_hash, status, created_at)`
- `roles(id, name)`
- `user_roles(user_id, role_id)`
- `companies(id, ticker, cik, name, exchange, sector)`
- `reports(id, tenant_id, company_id, report_type, fiscal_period, fiscal_year, source, s3_key, checksum, status, uploaded_by, uploaded_at, processed_at)`
- `report_pages(id, report_id, page_no, text_content, ocr_confidence)`
- `statements(id, report_id, statement_type, currency, unit_scale)`
- `line_items(id, statement_id, taxonomy_key, label, value, period_end, confidence, source_page)`
- `ratio_definitions(id, code, name, formula, category, enabled)`
- `ratio_results(id, company_id, report_id, ratio_code, value, computed_at)`
- `investment_scores(id, company_id, report_id, model_version, score, rationale_json)`
- `ai_reports(id, company_id, report_id, model, prompt_version, summary_md, risks_md, opportunities_md, created_at)`
- `jobs(id, job_type, reference_id, status, retries, error_message, started_at, finished_at)`
- `audit_logs(id, tenant_id, actor_id, action, resource_type, resource_id, payload_json, created_at)`

### Indexing / Constraints

- Unique: `(company_id, report_type, fiscal_year, fiscal_period)` on `reports`
- Unique: `(company_id, report_id, ratio_code)` on `ratio_results`
- Partial index on `jobs(status)` for active job polling
- GIN index on `to_tsvector('english', report_pages.text_content)` for full-text search.
- Query guidance:
  - Use `websearch_to_tsquery` for end-user keyword search UX.
  - Use `plainto_tsquery` for sanitized plain text input.
  - Use `to_tsquery` for advanced power-user boolean syntax.
- Tradeoff: full-text indexing improves search latency but increases index storage and INSERT/UPDATE cost.
- Tenant isolation baseline: enforce `tenant_id` predicates in all repository queries.
- Tenant isolation evolution: start with app-layer isolation for MVP, then enable PostgreSQL RLS for enterprise/regulatory tenants or multi-team admin environments.
- RLS tradeoff: better defense in depth with modest query-planning overhead from RLS policies.

---

## 7) Event-Driven Architecture

Events are immutable facts with `event_id`, `trace_id`, `tenant_id`, `occurred_at`, `schema_version`, `payload`.

```mermaid
flowchart LR
    U[Upload Accepted] --> E1[report.uploaded]
    E1 --> P[Document Processor]
    P --> E2[report.processed]
    E2 --> R[Ratio Worker]
    R --> E3[metrics.calculated]
    E3 --> A[AI Worker]
    A --> E4[analysis.completed]
    A --> E5[analysis.failed]
```

### Reliability Pattern

- Outbox table in PostgreSQL + publisher relay to RabbitMQ.
- Consumer idempotency table (`processed_events`) to deduplicate.
- Dead-letter queues for poison messages.
- Exponential retry with bounded attempts and alerting.

---

## 8) RabbitMQ Topology

### Exchanges

- `finsight.domain` (topic)
- `finsight.retry` (topic)
- `finsight.dlx` (topic)

### Queues and Routing Keys

- Queue `q.doc.process` ← `report.uploaded`
- Queue `q.metrics.calculate` ← `report.processed`
- Queue `q.ai.generate` ← `metrics.calculated`
- Queue `q.notifications` ← `analysis.completed`, `analysis.failed`
- Queue `q.audit` ← `*.created`, `*.updated`, `*.deleted`

### DLQ

- `q.doc.process.dlq`, `q.metrics.calculate.dlq`, `q.ai.generate.dlq`
- Route from primary queues via dead-letter policy after max retries.

---

## 9) Celery Worker Design

Worker pools:

1. `worker-ingestion` (I/O heavy): file integrity checks, metadata extraction.
2. `worker-docproc` (CPU heavy): PDF parsing + OCR.
3. `worker-metrics` (CPU/memory moderate): ratio and scoring engine.
4. `worker-ai` (network bound): OpenAI inference with rate limiting.
5. `worker-notify` (I/O bound): email/webhook notifications.

Recommended Celery config:

- `acks_late=True`, `task_reject_on_worker_lost=True`
- Separate queues per worker type
- Prefetch tuned by workload (`1` for docproc, higher for I/O workers)
- Soft/hard time limits for OCR and LLM calls
- Redis as Celery result backend (short retention only).

---

## 10) REST API Design (FastAPI)

### Auth & Users

- `POST /v1/auth/login`
- `POST /v1/auth/refresh`
- `POST /v1/auth/logout`
- `GET /v1/users/me`

### Companies & Reports

- `POST /v1/companies`
- `GET /v1/companies/{company_id}`
- `GET /v1/companies?ticker=...`
- `POST /v1/reports` (multipart upload)
- `GET /v1/reports/{report_id}`
- `GET /v1/reports/{report_id}/status`
- `GET /v1/companies/{company_id}/reports`

### Financial Data & Analytics

- `GET /v1/reports/{report_id}/statements`
- `GET /v1/reports/{report_id}/ratios`
- `GET /v1/companies/{company_id}/ratios/history`
- `POST /v1/reports/{report_id}/recalculate`

### AI Analysis

- `POST /v1/reports/{report_id}/ai-analysis`
- `GET /v1/reports/{report_id}/ai-analysis`
- `GET /v1/companies/{company_id}/investment-summary`

### Ops

- `GET /health/live`
- `GET /health/ready`
- `GET /metrics` (Prometheus scrape)

---

## 11) Authentication & Authorization

- **AuthN**: JWT access tokens + refresh tokens; optional SSO (OIDC/SAML for enterprise).
- **AuthZ**: RBAC with roles (`admin`, `analyst`, `viewer`) and tenant scoping.
- **Security controls**:
  - Password hashing with Argon2id.
    - Recommended baseline: `memory_cost=65536 KiB (~ 64 MiB)`, `time_cost=3`, `parallelism=4`.
    - Tune from benchmark results; target roughly sub-200ms verification on production hardware to balance security and UX.
    - Load-test under concurrent authentication to validate memory headroom.
  - Signed JWT keys in AWS KMS/Secrets Manager.
  - Short-lived access tokens + refresh token rotation.
  - API rate limiting (Redis token bucket).
  - Audit logs for privileged actions.

---

## 12) AWS Deployment Architecture

```mermaid
flowchart TB
    Route53[Route53] --> CF[CloudFront + WAF]
    CF --> ALB[Application Load Balancer]

    subgraph VPC
      ALB --> ECSAPI[ECS Fargate Service: FastAPI]
      ECSAPI --> ECSW[ECS Fargate Service: Celery Workers]
      ECSAPI --> RDS[(RDS PostgreSQL Multi-AZ)]
      ECSW --> RDS
      ECSAPI --> ElastiCache[(ElastiCache Redis)]
      ECSW --> ElastiCache
      ECSAPI --> MQ[(Amazon MQ RabbitMQ)]
      ECSW --> MQ
      ECSAPI --> S3[(S3 Reports Bucket)]
      ECSW --> S3
      ECSAPI --> OTEL[OpenTelemetry Collector]
      ECSW --> OTEL
    end

    OTEL --> Prom[Amazon Managed Prometheus]
    Prom --> Graf[Amazon Managed Grafana]
    ECSAPI --> Sentry[Sentry]
    ECSW --> Sentry
```

---

## 13) CI/CD Pipeline (GitHub Actions)

1. **PR pipeline**
   - Ruff/Black/isort/mypy
   - Pytest (unit + integration with Testcontainers)
   - Frontend lint/typecheck/tests
   - Trivy image scan + dependency scan
2. **Main branch pipeline**
   - Build multi-stage Docker images
   - Push to ECR
   - Run Alembic migration checks
   - Deploy to staging (ECS) via OIDC role
   - Smoke tests
3. **Production promotion**
   - Manual approval gate
   - Blue/green or canary release
   - Automatic rollback based on SLO alarms.

---

## 14) Monitoring, Logging, Tracing

- **Metrics**: Prometheus + custom business metrics (processing latency, extraction confidence, queue depth).
- **Tracing**: OpenTelemetry traces from API → RabbitMQ → Celery → OpenAI.
- **Logs**: Structured JSON logs shipped to CloudWatch/OpenSearch.
- **Errors**: Sentry for backend/frontend exception tracking.
- **SLO examples**:
  - P95 upload-to-analysis completion < 10 min.
    - Completion boundary: from successful `POST /v1/reports` acceptance to persisted `analysis.completed` event with AI report stored.
    - Scope: includes queue wait + processing time under normal load for standard-size filings; define separate large-document SLO tiers.
  - API error rate < 1%
  - Queue lag < 2 min for standard plan.

---

## 15) Clean Architecture + DDD Application

- Domain models independent of FastAPI/ORM/queue frameworks.
- Use cases expose ports; adapters implement ports.
- Each bounded context has its own module and anti-corruption layer.
- Shared kernel only for strongly common concepts (Money, FiscalPeriod, TenantId).
- Domain events emitted from aggregate roots and published through outbox.

---

## 16) Major Tradeoffs and Alternatives

1. **RabbitMQ + Celery vs Kafka + Faust/Streams**
   - Chosen for Python ecosystem maturity and operational simplicity.
   - Kafka better for very high-throughput replay/event-sourcing but adds ops complexity.

2. **PostgreSQL vs DynamoDB**
   - PostgreSQL chosen for relational integrity, SQL analytics, and strong transactional behavior.
   - DynamoDB could improve extreme-scale write throughput but complicates joins/reporting.

3. **PydanticAI + OpenAI vs self-hosted models**
   - OpenAI accelerates quality/time-to-market.
   - Self-hosted models reduce vendor lock-in and data exposure but require MLOps investment.

4. **ECS Fargate vs EKS**
   - Fargate minimizes ops overhead initially.
   - EKS preferred if needing deeper scheduling control and lower large-scale compute costs.

5. **PyMuPDF/pdfplumber + OCR vs external document AI**
   - In-house pipeline gives transparency and lower unit cost at scale.
   - Managed OCR/Document AI can increase extraction accuracy for complex tables with higher cost.

---

## 17) Additional Mermaid: Processing State Machine

```mermaid
stateDiagram-v2
    [*] --> UPLOADED
    UPLOADED --> PROCESSING: report.uploaded consumed
    PROCESSING --> PROCESSED: extraction success
    PROCESSING --> FAILED: extraction fatal error
    PROCESSED --> METRICS_READY: ratios computed
    METRICS_READY --> AI_READY: AI report generated
    AI_READY --> [*]
    FAILED --> [*]
```

---

## 18) Scaling Strategy to 1M Users

### Phase 1 (0–50k users)
- Single region, Multi-AZ RDS, autoscaled ECS services.
- Queue partitioning by task type.

### Phase 2 (50k–300k users)
- Read replicas for analytics queries.
- Redis caching for hot ratios and company summaries.
- Separate worker fleets by tenant tier.

### Phase 3 (300k–1M users)
- Multi-region active/passive for DR.
- Shard data by tenant segment or company universe.
- Introduce event streaming backbone (Kafka) for replay-heavy analytics.
- Dedicated model gateway with batching and budget controls.
- Add asynchronous API patterns (webhooks + SSE for job completion).

### Cross-phase controls
- Strict cost observability per tenant/job.
- Tenant-aware rate limits and quotas.
- SLO-driven autoscaling (queue lag + CPU + p95 latency).

---

## Testing Strategy (Pytest + Testcontainers)

- Unit tests for domain services (ratio formulas, scoring rules).
- Integration tests for SQLAlchemy repositories against PostgreSQL container.
- End-to-end async tests with RabbitMQ + Redis + Celery workers in Testcontainers.
- Contract tests for OpenAI responses using Pydantic schemas + mocked API.

This architecture is implementation-ready for a senior engineering team and aligned with the specified technology constraints.
