# Technology Stack

## Frontend

| Technology | Role |
|-|-|
| **React 19** | UI framework |
| **TypeScript** | Type safety across the entire frontend |
| **Vite** | Build tool and dev server |
| **Tailwind CSS** | Utility-first styling |
| **shadcn/ui** | Accessible, composable UI component library |
| **React Router** | Client-side routing between views |
| **TanStack Query** | Server state management, API caching, background refetching |
| **React Hook Form** | Performant form handling |
| **Zod** | Schema validation for forms and API responses |
| **Framer Motion** | Animations — tension breathing, shake effects on conflict, contextual zoom in Replay Mode |
| **Recharts** | Charts and data visualizations for insights, telemetry panel, and heatmap |

Served locally via the Vite dev server (or a static build served by any local web server). Views include Analysis, Replay Mode, Live Telemetry, and Audit.

---

## Backend

| Technology | Role |
|-|-|
| **Python 3.13** | Primary language — dominant in AI, NLP, and model experimentation |
| **FastAPI** | Async REST API framework — fits well for exposing AI services |
| **Pydantic v2** | Data validation, serialization, and structured LLM outputs |
| **SQLAlchemy 2.0** | ORM for PostgreSQL |
| **Alembic** | Database schema migrations |
| **Uvicorn** | ASGI server |

Runs locally via **Uvicorn**, optionally containerized with Docker.

---

## AI Layer

| Technology | Role |
|-|-|
| **Ollama** | Local inference server — runs `llama3.2:3b` and `nomic-embed-text` on-device, no external API |
| **llama3.2:3b** | Crisis classification (LLM call #1) + pragmatic analysis (LLM call #2) |
| **nomic-embed-text** | Local embedding model for document and query vectorization |
| **LangGraph** | Orchestration of the governance pipeline flow — directed graph execution with short-circuit logic |
| **LangChain** | Selective use — only where it adds value (e.g., document loaders for indexing) |
| **Structured Outputs** | Typed JSON responses validated by Pydantic v2 |

Two LLM calls per message:
1. **Crisis classifier** — specialized, minimalist prompt (~150-200 tokens). Runs in parallel with retrieval, but remains strictly blocking.
2. **Pragmatic analysis** — receives message, context, and reranked fragments with mandatory grounding.

The analysis call dominates both latency and token consumption per request, so it is the correct target for any optimization work. Exact latency depends on local hardware (GPU/VRAM) — see [AI Pipeline](ai-pipeline.md) for the benchmarking note.

---

## RAG

| Technology | Role |
|-|-|
| **PostgreSQL + pgvector** | Vector storage and similarity search (HNSW index) |
| **PostgreSQL `tsvector` + GIN** | Lexical search branch |
| **Ollama (`nomic-embed-text`)** | Document chunk vectorization, running locally |
| **Chunking pipeline** | Document splitting for optimal retrieval |
| **Reciprocal Rank Fusion** | Merges the vector and lexical ranked lists (`k = 60`) |
| **Cross-encoder reranker** | Joint `(query, chunk)` scoring; produces the calibrated value that gates generation |

pgvector provides vector similarity only — it has no built-in hybrid retrieval or reranker, so that layer is implemented explicitly rather than consumed as a managed feature. **Hybrid search here is implemented, not consumed.** See [AI Pipeline](ai-pipeline.md) for the SQL and the fusion logic.

The corpus is configurable: any compatible document collection can be indexed without modifying governance logic.

---

## Data

| Technology | Role |
|-|-|
| **PostgreSQL (Docker container)** | Primary database — traces, prompt versions, audit records |
| **pgvector** | Vector extension for embeddings and semantic search |
| **Redis** | Cache layer and session management |

All traces are stored with a unique `trace_id` and serve as the single source of truth for telemetry, Replay Mode, and the live panel.

Raw message text and derived annotations are stored in separate columns so that erasure requests and retention TTLs can purge personal data without destroying the audit trail.

---

## Local Infrastructure

| Component | Role |
|-|-|
| **Docker Compose** | Orchestrates PostgreSQL, Redis, and optionally backend/frontend containers |
| **Ollama** | Local inference server, run natively on the host for direct GPU access |
| **Local filesystem** | Document storage for the RAG corpus |

---

## Security

| Technology | Role |
|-|-|
| **JWT** | Token-based authentication, self-issued by the backend |
| **Rate limiting** | Per user/IP request throttling |
| **Prompt injection detection** | Pipeline-level policy check |

**No standing credentials to leak.** Since inference, embeddings, database, and cache all run on the local machine, there is no third-party API key in configuration and no service-to-service cloud auth to manage. Nothing needs to be rotated because nothing is a shared secret.

Additional security layers: rate limiting per user/IP, prompt injection detection, and emotional crisis classification — all coded in the pipeline, not in the prompt.

---

## Communication

| Technology | Role |
|-|-|
| **REST API** | Primary communication between frontend and backend |
| **WebSockets** | Real-time analysis updates and live telemetry panel |
| **Background Tasks** | Async processing for evaluation jobs |
| **Celery + Redis** | Task queue for long-running processes |

---

## Observability

| Technology | Role |
|-|-|
| **OpenTelemetry** | Distributed tracing standard (console exporter, optional local Jaeger) |
| **Structured Logging** | Consistent, queryable log format |
| **Health Checks** | `/health` endpoint monitoring all connected local services |

---

## DevOps

| Technology | Role |
|-|-|
| **Docker** | Container runtime |
| **Docker Compose** | Multi-container local development |
| **GitHub Actions** | CI pipeline — lint, test, automated evaluation (no deploy target) |
| **Ruff** | Python linter and formatter (replaces flake8 + isort) |
| **Black** | Python code formatter |
| **Pytest** | Test framework |
| **pre-commit** | Git hooks for code quality enforcement |
