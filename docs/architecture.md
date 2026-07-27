# Architecture

## System Overview

Production architecture on AWS, with clear separation between frontend, backend, and AI services. The backend is built with **Python 3.13 and FastAPI**, the frontend with **React 19**, and the AI layer is served through **OpenRouter** (`gpt-4.1` for generation, `text-embedding-3-large` for embeddings) with **PostgreSQL + pgvector** (Amazon RDS) for data and vector search.

There are no standing AWS credentials in configuration — every AWS-native call (RDS, S3, Secrets Manager, CloudWatch) is authenticated via an IAM role assumed by the running service. The one exception is OpenRouter, a third party outside the AWS trust boundary, whose API key is resolved from Secrets Manager at startup rather than stored anywhere. See [Cloud Deployment (AWS)](cloud-deployment-aws.md) for the full rationale.

---

## High-Level Architecture

```
┌──────────────────────────────────────────────────────────────┐
│                      USER (browser)                          │
└────────────────────────┬─────────────────────────────────────┘
                         │ HTTPS
┌────────────────────────▼─────────────────────────────────────┐
│         AWS Amplify Hosting (Frontend React 19)              │
│  TypeScript · Tailwind CSS · shadcn/ui · Recharts            │
│  Framer Motion · TanStack Query · React Hook Form · Zod      │
└────────────────────────┬─────────────────────────────────────┘
                         │ REST API / WebSockets
                         │ (Cognito-issued OAuth2 token)
┌────────────────────────▼─────────────────────────────────────┐
│    AWS App Runner / ECS Fargate — FastAPI Backend             │
│                                                              │
│  Governance pipeline (cascade short-circuit):                │
│                                                              │
│  1. Policy validation (no LLM)                               │
│     → min length, language, rate limit, prompt injection     │
│                    ↓                                         │
│  2. Parallel branch                                          │
│     ├─ Crisis classifier (LLM call #1)   [blocking]          │
│     └─ Embedding + hybrid search (vector + lexical + RRF)    │
│                    ↓                                         │
│  3. Cross-encoder reranking + confidence gate                │
│     → calibrated score threshold; blocks if ungrounded       │
│                    ↓                                         │
│  4. Main LLM — Pragmatic analysis (LLM call #2)              │
│     → OpenRouter (gpt-4.1) with mandatory grounding          │
│                    ↓                                         │
│  5. Traceability                                             │
│     → Full record in RDS PostgreSQL with unique trace_id     │
│                                                              │
│  Orchestration: LangGraph                                    │
│  Validation: Pydantic v2 + Structured Outputs                │
│  Auth: IAM role (AWS services) + Secrets Manager (OpenRouter)│
└──┬───────────┬──────────────────┬────────────────────────────┘
   │           │                  │
   ▼           ▼                  ▼
OpenRouter    Amazon RDS         ElastiCache
gpt-4.1       PostgreSQL         for Redis
text-embed-   + pgvector         (Cache, sessions)
3-large       HNSW + GIN index
              Traces, vectors,
              prompt versions
                    │
                    ▼
             CloudWatch + AWS X-Ray + OpenTelemetry
                    │
                    ▼
             GitHub Actions (CI/CD: lint + test + deploy, OIDC to AWS)
```

**Cascade short-circuit pipeline:** if any policy triggers at any step, the flow stops and the main model is never invoked. Order is strict: policies → crisis/retrieval → confidence gate → pragmatic analysis → traceability.

**Why step 2 is parallel.** Crisis detection is strictly blocking, but it does not depend on retrieval and retrieval does not depend on it. Running them concurrently removes latency from every request that passes the check — the vast majority — without weakening the guarantee. If the classifier fires, the retrieval result is discarded and the request short-circuits with a `422`.

---

## Frontend

React 19 + TypeScript + Vite, built as a static bundle and served through **AWS Amplify Hosting**, which builds and deploys on every push to the tracked branch.

| Technology | Role |
|-|-|
| **Tailwind CSS** | Utility-first styling |
| **shadcn/ui** | Accessible, composable UI components |
| **React Router** | Client-side routing |
| **TanStack Query** | Server state management and API caching |
| **React Hook Form + Zod** | Form handling with schema validation |
| **Framer Motion** | Animations (tension breathing, shake effects, contextual zoom) |
| **Recharts** | Charts and data visualizations for insights and telemetry |

Main views:
- **Analysis** — Send messages and receive pragmatic analysis.
- **Replay Mode** — Interactive timeline of analyzed conversations.
- **Live Telemetry** — Real-time session panel with emotional velocity, conflict acceleration, accumulated tension.
- **Audit** — Trace lookup by `trace_id`.

---

## Backend

Python 3.13 + FastAPI, containerized with Docker and deployed to **AWS App Runner** (default) or **ECS Fargate** (see [Deployment](deployment.md)).

| Technology | Role |
|-|-|
| **FastAPI** | Async REST API framework |
| **Pydantic v2** | Data validation, serialization, structured outputs |
| **SQLAlchemy 2.0** | ORM for PostgreSQL |
| **Alembic** | Database migrations |
| **Uvicorn** | ASGI server |
| **LangGraph** | Orchestration of the governance pipeline flow |
| **Celery + Redis** | Background tasks for long-running processes |

Main pipeline components:

| Module | Responsibility |
|-|-|
| `policy_engine.py` | Policy validation without LLM |
| `crisis_classifier.py` | Emotional crisis classifier (LLM call #1) |
| `retrieval_service.py` | Hybrid search — vector + lexical branches, RRF fusion |
| `rerank_service.py` | Cross-encoder reranking and confidence gate |
| `llm_service.py` | Main pragmatic analysis (LLM call #2) |
| `trace_store.py` | Trace persistence with `trace_id` |
| `retention_job.py` | Scheduled purge of expired raw message text |

---

## RAG Layer

**PostgreSQL + pgvector** (Amazon RDS) with explicitly implemented hybrid retrieval.

pgvector provides vector similarity only, so hybrid search is built in three stages rather than consumed as a feature:

1. **Vector branch** — cosine distance over an HNSW index
2. **Lexical branch** — `ts_rank_cd` over a GIN-indexed `tsvector` column
3. **Fusion** — Reciprocal Rank Fusion (`k = 60`) merging both ranked lists

A **cross-encoder reranker** then scores the fused candidates jointly and produces the calibrated value that gates generation. The RRF score itself is rank-based and is never used as a confidence threshold.

Documents are chunked and vectorized via **OpenRouter** (`text-embedding-3-large`, 3072-dim). The corpus is **swappable and configurable**: the retrieval architecture is not coupled to any source, so any compatible document collection can be indexed without touching governance logic.

> Full implementation detail, including the SQL and the rationale for the threshold design, is in [AI Pipeline](ai-pipeline.md).

---

## Data Layer

| Technology | Role |
|-|-|
| **Amazon RDS for PostgreSQL** | Primary database — traces, prompt versions, audit records |
| **pgvector** | Vector extension for embeddings and semantic search |
| **Amazon ElastiCache for Redis** | Cache layer and session management |

Each interaction generates a persistent record identified by a unique `trace_id` that includes: input message and context, retrieved fragments with vector/lexical/RRF/reranker scores, embedding and reranker model identity, exact prompt and version (`prompt_version`), model version, confidence level, applied policy (if any), `grounded` field, per-stage latency, and timestamp.

These traces are the **only data source** for conversational telemetry, the live panel, and Replay Mode.

### Split storage for erasure

Raw message text and derived annotations live in separate columns. The annotation layer — scores, policy decisions, model versions, intent and emotion labels — constitutes the audit record; the raw text is the personal data.

This separation allows an erasure request to null the text while leaving the audit trail intact and reconstructible. Replay Mode continues to work with the content redacted. Raw text also carries a configurable TTL (default 90 days) enforced by a scheduled purge job.

---

## AWS Infrastructure

| Component | Role |
|-|-|
| **AWS App Runner / ECS Fargate** | Runs the FastAPI backend container, scaling on request volume |
| **AWS Amplify Hosting** | Builds and serves the React frontend from the tracked Git branch |
| **Amazon RDS for PostgreSQL + pgvector** | Traces, vectors, audit records — single instance, single consistency model |
| **Amazon ElastiCache for Redis** | Cache and session storage |
| **Amazon S3** | Document storage for the RAG corpus |
| **AWS Secrets Manager** | Holds the one standing credential the stack needs — the OpenRouter API key |
| **IAM roles** | Every AWS-native call (RDS, S3, Secrets Manager, CloudWatch) authenticates with short-lived, automatically rotated credentials — nothing to leak through logs or environment dumps |

See [Cloud Deployment (AWS)](cloud-deployment-aws.md) for region configuration and data residency notes.

---

## Security

| Mechanism | Implementation |
|-|-|
| **User authentication** | Amazon Cognito, OAuth2 tokens validated by the backend |
| **Database auth** | RDS IAM authentication — short-lived tokens, no stored password |
| **Rate limiting** | Per user/IP (> 10 req/min) |
| **Prompt injection** | Detection and audit logging |
| **Crisis classification** | Dedicated LLM classifier |
| **Secrets** | AWS Secrets Manager holds only the OpenRouter API key — every AWS-native credential is IAM-issued and short-lived |
| **CI/CD** | GitHub Actions authenticates to AWS via OIDC federated credentials — no long-lived IAM access key in GitHub |

Since AWS-native services authenticate via IAM roles, the only credential that can leak through logs or environment dumps is the OpenRouter key — and it lives in Secrets Manager, resolved at runtime rather than set in configuration.

---

## Observability

| Layer | Technology |
|-|-|
| **Distributed tracing** | OpenTelemetry, exported to AWS X-Ray |
| **Metrics** | Amazon CloudWatch / `GET /metrics` |
| **Logging** | Structured logging, shipped to CloudWatch Logs |
| **Health checks** | `/health` endpoint monitoring all connected AWS services and the OpenRouter API |

CloudWatch aggregates metrics from both live interactions and the evaluation job, enabling behavior monitoring and quantitative prompt version comparison.

**Capacity signal worth watching:** OpenRouter's rate limit and per-token cost, not local hardware. See [Deployment](deployment.md) for the capacity model.

---

## Conversational Telemetry

SubtextAI can be understood as a **telemetry engine applied to human conversation**. Just as mechanical system telemetry records and formalizes continuous signals, SubtextAI formalizes dialogue dynamics from signals it already computes and persists per message.

These metrics are not a speculative layer or a new model: they are a formalization of data that already exists in the traces. They inherit the same grounding and traceability by `trace_id` as the rest of the system.

| Metric | Definition | Derivation |
|-|-|-|
| **Emotional velocity** | Variation of emotional intensity between two consecutive messages (first derivative). | From per-message annotated intensity. |
| **Conflict acceleration** | Variation of emotional velocity (second derivative). Whether escalation is accelerating or decelerating. | From the emotional velocity series. |
| **Conversational friction** | Accumulated tension between explicit and implicit meaning throughout the dialogue. | From subtext and tension already recorded. |
| **Intent curves** | Trajectory of intent changes (e.g., `neutral → defensive → aggressive`) across the conversation. | From per-message detected intent. |

Because they derive from the annotation layer rather than the raw text, these metrics survive text-level erasure intact.

---

## Replay Mode — Architecture

Replay Mode consumes the `GET /replay/{trace_id}` endpoint and reconstructs from already-persisted traces, with no new model calls during playback. Loads in under 2 seconds for typical conversations.

It is a frontend layer (React 19) operating on persisted traces. Pre-recorded conversations in Demo Mode are real, previously analyzed and stored traces.

---

## Live Telemetry — Architecture

Since it relies on existing endpoints (`/analyze` per message), the live telemetry view requires no new streaming infrastructure: it is a frontend aggregation layer over already-delivered and audited responses. Each data point in the live panel corresponds 1:1 to a response that has already passed through the complete governance pipeline.

WebSocket connections can optionally provide real-time updates to the telemetry panel.

---

## Folder Structure

```
subtext-ai/
│
├── frontend/                            # React 19 application
│   ├── src/
│   │   ├── components/                  # shadcn/ui + custom components (includes live telemetry panel)
│   │   ├── pages/                       # Views (Analysis, Replay, Telemetry, Audit)
│   │   ├── services/                    # API integration (TanStack Query)
│   │   ├── hooks/                       # Custom React hooks
│   │   ├── types/                       # TypeScript types for backend responses
│   │   └── lib/                         # Utilities (Zod schemas, etc.)
│   ├── .env.example
│   ├── tailwind.config.ts
│   └── vite.config.ts
│
├── backend/                             # FastAPI application
│   ├── api/                             # FastAPI routes (analyze, evaluate, replay, audit, metrics, prompts, health)
│   ├── domain/                          # Domain models (Pydantic v2)
│   ├── application/                     # Business logic, use cases
│   ├── infrastructure/                  # Database, OpenRouter client, AWS SDK clients
│   ├── ai/                              # LLM integration, crisis classifier
│   ├── rag/                             # Retrieval, RRF fusion, reranking, embeddings
│   ├── prompts/                         # Versioned system prompts
│   ├── migrations/                      # Alembic migrations
│   └── main.py                          # FastAPI app entry point
│
├── docker/                              # Container configuration
│   ├── Dockerfile.backend
│   ├── Dockerfile.frontend
│   └── docker-compose.yml               # Local dev services only (Postgres/Redis test doubles)
│
├── scripts/                             # Indexing and evaluation utilities
│   ├── index_documents.py
│   └── evaluation_harness.py
│
├── tests/                               # Test suite
│   ├── unit/
│   ├── integration/
│   └── conftest.py
│
├── docs/                                # Project documentation
│
├── .github/
│   └── workflows/                       # GitHub Actions CI/CD (lint, test, deploy to AWS)
│
├── .env.example
├── pyproject.toml
└── README.md
```
