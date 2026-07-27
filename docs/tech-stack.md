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

Built as a static bundle and served in production via **AWS Amplify Hosting**. Views include Analysis, Replay Mode, Live Telemetry, and Audit.

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

Containerized with Docker and deployed to **AWS App Runner** (default) or **ECS Fargate**.

---

## AI Layer

| Technology | Role |
|-|-|
| **OpenRouter** | Single API and billing surface for both models — no direct provider account, swapping models is a config change |
| **gpt-4.1** (via OpenRouter) | Crisis classification (LLM call #1) + pragmatic analysis (LLM call #2) |
| **text-embedding-3-large** (via OpenRouter) | Embedding model for document and query vectorization (3072-dim) |
| **LangGraph** | Orchestration of the governance pipeline flow — directed graph execution with short-circuit logic |
| **LangChain** | Selective use — only where it adds value (e.g., document loaders for indexing) |
| **Structured Outputs** | Typed JSON responses validated by Pydantic v2 |

Two LLM calls per message:
1. **Crisis classifier** — specialized, minimalist prompt (~150-200 tokens). Runs in parallel with retrieval, but remains strictly blocking.
2. **Pragmatic analysis** — receives message, context, and reranked fragments with mandatory grounding.

The analysis call dominates both latency and token consumption per request, so it is the correct target for any optimization work. Latency depends on OpenRouter and the underlying model provider rather than local hardware — see [AI Pipeline](ai-pipeline.md) for the benchmarking note.

---

## RAG

| Technology | Role |
|-|-|
| **PostgreSQL + pgvector** | Vector storage and similarity search (HNSW index) |
| **PostgreSQL `tsvector` + GIN** | Lexical search branch |
| **OpenRouter (`text-embedding-3-large`)** | Document chunk vectorization |
| **Chunking pipeline** | Document splitting for optimal retrieval |
| **Reciprocal Rank Fusion** | Merges the vector and lexical ranked lists (`k = 60`) |
| **Cross-encoder reranker** | Joint `(query, chunk)` scoring; produces the calibrated value that gates generation |

pgvector provides vector similarity only — it has no built-in hybrid retrieval or reranker, so that layer is implemented explicitly rather than consumed as a managed feature. **Hybrid search here is implemented, not consumed.** See [AI Pipeline](ai-pipeline.md) for the SQL and the fusion logic.

The corpus is configurable: any compatible document collection can be indexed without modifying governance logic.

---

## Data

| Technology | Role |
|-|-|
| **Amazon RDS for PostgreSQL** | Primary database — traces, prompt versions, audit records |
| **pgvector** | Vector extension for embeddings and semantic search |
| **Amazon ElastiCache for Redis** | Cache layer and session management |

All traces are stored with a unique `trace_id` and serve as the single source of truth for telemetry, Replay Mode, and the live panel.

Raw message text and derived annotations are stored in separate columns so that erasure requests and retention TTLs can purge personal data without destroying the audit trail.

---

## AWS Infrastructure

| Component | Role |
|-|-|
| **AWS App Runner / ECS Fargate** | Runs the FastAPI backend container |
| **AWS Amplify Hosting** | Builds and serves the React frontend |
| **Amazon S3** | Document storage for the RAG corpus |
| **AWS Secrets Manager** | Holds the OpenRouter API key |
| **IAM roles** | Authenticate every AWS-native call with short-lived, automatically rotated credentials |

---

## Security

| Technology | Role |
|-|-|
| **Amazon Cognito** | User authentication and OAuth2 token issuance |
| **IAM roles** | Service-to-service auth for RDS, S3, Secrets Manager, CloudWatch — no standing credential |
| **AWS Secrets Manager** | Holds the one credential IAM can't cover: the OpenRouter API key |
| **Rate limiting** | Per user/IP request throttling |
| **Prompt injection detection** | Pipeline-level policy check |

**One standing credential, tightly scoped.** Every AWS-native call authenticates via IAM role — short-lived, automatically rotated, nothing to leak. The one exception is OpenRouter, a third party outside the AWS trust boundary: its API key lives in Secrets Manager and is resolved at runtime, never stored in configuration or source.

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
| **OpenTelemetry** | Distributed tracing standard, exported to AWS X-Ray |
| **Structured Logging** | Consistent, queryable log format, shipped to CloudWatch Logs |
| **Health Checks** | `/health` endpoint monitoring all connected AWS services and OpenRouter |

---

## DevOps

| Technology | Role |
|-|-|
| **Docker** | Container runtime and image build for App Runner / ECS |
| **AWS App Runner / ECS Fargate** | Backend deployment target |
| **AWS Amplify Hosting** | Frontend deployment target |
| **GitHub Actions** | CI/CD pipeline — lint, test, build, deploy to AWS via OIDC federated credentials |
| **Ruff** | Python linter and formatter (replaces flake8 + isort) |
| **Black** | Python code formatter |
| **Pytest** | Test framework |
| **pre-commit** | Git hooks for code quality enforcement |
