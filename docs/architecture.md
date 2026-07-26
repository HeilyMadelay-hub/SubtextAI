# Architecture

## System Overview

Fully local architecture with clear separation between frontend, backend, and AI services. The backend is built with **Python 3.13 and FastAPI**, the frontend with **React 19**, and the AI layer runs on **Ollama** (local inference) with **PostgreSQL + pgvector** for data and vector search.

Everything runs on the developer's machine. There are no API keys, no cloud account, and no external service calls — model inference, embeddings, database, and cache all execute locally via Docker and Ollama.

---

## High-Level Architecture

```
┌──────────────────────────────────────────────────────────────┐
│                      USER (browser)                          │
└────────────────────────┬─────────────────────────────────────┘
                         │ HTTPS
┌────────────────────────▼─────────────────────────────────────┐
│          Local Dev Server — Vite (Frontend React 19)         │
│  TypeScript · Tailwind CSS · shadcn/ui · Recharts            │
│  Framer Motion · TanStack Query · React Hook Form · Zod      │
└────────────────────────┬─────────────────────────────────────┘
                         │ REST API / WebSockets
                         │ (JWT, self-issued)
┌────────────────────────▼─────────────────────────────────────┐
│      Local FastAPI Backend (Uvicorn, optional Docker)        │
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
│     → Ollama (llama3.2:3b) with mandatory grounding          │
│                    ↓                                         │
│  5. Traceability                                             │
│     → Full record in PostgreSQL with unique trace_id         │
│                                                              │
│  Orchestration: LangGraph                                    │
│  Validation: Pydantic v2 + Structured Outputs                │
│  No API keys, no external calls                             │
└──┬───────────┬──────────────────┬────────────────────────────┘
   │           │                  │
   ▼           ▼                  ▼
Ollama        PostgreSQL         Redis
(local)       (Docker)           (Cache, sessions)
llama3.2:3b   + pgvector
nomic-embed-  HNSW + GIN index
text          Traces, vectors,
               prompt versions
                    │
                    ▼
             Local structured logs + OpenTelemetry
             (console exporter)
                    │
                    ▼
             GitHub Actions (CI: lint + test, evaluation harness)
```

**Cascade short-circuit pipeline:** if any policy triggers at any step, the flow stops and the main model is never invoked. Order is strict: policies → crisis/retrieval → confidence gate → pragmatic analysis → traceability.

**Why step 2 is parallel.** Crisis detection is strictly blocking, but it does not depend on retrieval and retrieval does not depend on it. Running them concurrently removes roughly 140 ms from every request that passes the check — the vast majority — without weakening the guarantee. If the classifier fires, the retrieval result is discarded and the request short-circuits with a `422`.

---

## Frontend

React 19 + TypeScript + Vite, served locally by the Vite dev server (or a static build served by any local web server).

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

Python 3.13 + FastAPI, run locally via **Uvicorn**, optionally containerized with Docker (see [Deployment](deployment.md)).

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

**PostgreSQL + pgvector** with explicitly implemented hybrid retrieval.

pgvector provides vector similarity only, so hybrid search is built in three stages rather than consumed as a feature:

1. **Vector branch** — cosine distance over an HNSW index
2. **Lexical branch** — `ts_rank_cd` over a GIN-indexed `tsvector` column
3. **Fusion** — Reciprocal Rank Fusion (`k = 60`) merging both ranked lists

A **cross-encoder reranker** then scores the fused candidates jointly and produces the calibrated value that gates generation. The RRF score itself is rank-based and is never used as a confidence threshold.

Documents are chunked and vectorized locally with **Ollama** (`nomic-embed-text`). The corpus is **swappable and configurable**: the retrieval architecture is not coupled to any source, so any compatible document collection can be indexed without touching governance logic.

> Full implementation detail, including the SQL and the rationale for the threshold design, is in [AI Pipeline](ai-pipeline.md).

---

## Data Layer

| Technology | Role |
|-|-|
| **PostgreSQL (Docker container)** | Primary database — traces, prompt versions, audit records |
| **pgvector** | Vector extension for embeddings and semantic search |
| **Redis** | Cache layer and session management |

Each interaction generates a persistent record identified by a unique `trace_id` that includes: input message and context, retrieved fragments with vector/lexical/RRF/reranker scores, embedding and reranker model identity, exact prompt and version (`prompt_version`), model version, confidence level, applied policy (if any), `grounded` field, per-stage latency, and timestamp.

These traces are the **only data source** for conversational telemetry, the live panel, and Replay Mode.

### Split storage for erasure

Raw message text and derived annotations live in separate columns. The annotation layer — scores, policy decisions, model versions, intent and emotion labels — constitutes the audit record; the raw text is the personal data.

This separation allows an erasure request to null the text while leaving the audit trail intact and reconstructible. Replay Mode continues to work with the content redacted. Raw text also carries a configurable TTL (default 90 days) enforced by a scheduled purge job.

---

## Local Infrastructure

| Component | Role |
|-|-|
| **Docker Compose** | Orchestrates PostgreSQL, Redis, and (optionally) the backend and frontend containers |
| **Ollama** | Local inference server for `llama3.2:3b` and `nomic-embed-text` — runs natively on the host to use the GPU directly |
| **PostgreSQL + pgvector** | Traces, vectors, audit records — single container, single consistency model |
| **Redis** | Cache and session storage |
| **Local filesystem** | Document storage for the RAG corpus |

Nothing leaves the machine. There is no region to configure and no data residency question to resolve — all processing happens on the developer's own hardware.

---

## Security

| Mechanism | Implementation |
|-|-|
| **User authentication** | JWT, self-issued by the backend — no external identity provider needed |
| **Database auth** | Local password via Docker Compose secret |
| **Rate limiting** | Per user/IP (> 10 req/min) |
| **Prompt injection** | Detection and audit logging |
| **Crisis classification** | Dedicated LLM classifier |
| **Secrets** | None required — no third-party API keys exist in this stack |
| **CI** | GitHub Actions runs lint and tests only; no deploy target, no cloud credentials in CI |

Since inference, embeddings, and storage all run locally, there is no service-to-service cloud auth to manage and no API key that could leak through logs or environment dumps.

---

## Observability

| Layer | Technology |
|-|-|
| **Distributed tracing** | OpenTelemetry (console exporter, or optional local Jaeger) |
| **Metrics** | Local structured log file / `GET /metrics` |
| **Logging** | Structured logging |
| **Health checks** | `/health` endpoint monitoring all connected local services |

The local metrics endpoint aggregates both live interactions and the evaluation job, enabling behavior monitoring and quantitative prompt version comparison without any external monitoring service.

**Capacity signal worth watching:** local GPU/VRAM throughput, not a token quota. Ollama serves one generation at a time per GPU, so the real ceiling is hardware, not billing. See [Deployment](deployment.md) for the capacity model.

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
│   ├── infrastructure/                  # Database, Ollama client
│   ├── ai/                              # LLM integration, crisis classifier
│   ├── rag/                             # Retrieval, RRF fusion, reranking, embeddings
│   ├── prompts/                         # Versioned system prompts
│   ├── migrations/                      # Alembic migrations
│   └── main.py                          # FastAPI app entry point
│
├── docker/                              # Container configuration
│   ├── Dockerfile.backend
│   ├── Dockerfile.frontend
│   └── docker-compose.yml
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
│   └── workflows/                       # GitHub Actions CI (lint + test)
│
├── .env.example
├── pyproject.toml
└── README.md
```
