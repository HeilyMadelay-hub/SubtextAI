# Deployment Guide

## Prerequisites

- Python 3.13+
- Node.js 18+ and npm
- Docker and Docker Compose
- [Ollama](https://ollama.com) installed, with `llama3.2:3b` and `nomic-embed-text` pulled
- A GPU with enough VRAM to run `llama3.2:3b` comfortably (4GB+ recommended); CPU-only works but is slower

---

## Clone the repository

```bash
git clone https://github.com/<your-user>/subtextai.git
cd subtextai
```

---

## Docker Setup (local development)

The simplest way to run the full stack locally. Docker Compose provisions PostgreSQL with the pgvector extension and Redis alongside the application:

```bash
cp .env.example .env
# Edit .env with your local configuration (see Environment Variables below)

docker compose up --build
# Backend  → http://localhost:8000
# Frontend → http://localhost:5173
```

Ollama runs natively on the host (not inside Docker) so it can access the GPU directly. Make sure it is running and the required models are pulled before starting the backend:

```bash
ollama pull llama3.2:3b
ollama pull nomic-embed-text
ollama serve          # usually already running as a background service
```

The backend talks to Ollama over `http://localhost:11434` — no keys, no login, no cloud account.

---

## No Cloud Authentication Required

**The application makes zero calls to any external service.** There are no API keys anywhere in the codebase or configuration, because there is no third party to authenticate against — inference, embeddings, database, and cache all run on the local machine.

This removes an entire category of concerns that a cloud-backed stack would otherwise need: no credential rotation, nothing to leak through logs or environment dumps, and no service-to-service auth to configure.

### Backend client setup

```python
import ollama

client = ollama.AsyncClient(host="http://localhost:11434")

response = await client.chat(
    model="llama3.2:3b",
    messages=[{"role": "user", "content": prompt}],
    format="json",
)
```

---

## Database Setup

### Enable the pgvector extension

Inside the local PostgreSQL container:

```sql
CREATE EXTENSION IF NOT EXISTS vector;
```

### Index configuration

Alembic migrations create an **HNSW** index for vector search and a **GIN** index for lexical search:

```sql
CREATE INDEX ON document_chunks
    USING hnsw (embedding vector_cosine_ops)
    WITH (m = 16, ef_construction = 64);

CREATE INDEX ON document_chunks USING GIN (content_tsv);
```

**HNSW over IVFFlat.** HNSW gives better recall at equivalent latency and does not require a populated table at build time, which matters because the corpus is swappable. The trade-offs are a slower build and a larger index footprint — acceptable at this corpus size.

Query-time recall is tuned with `hnsw.ef_search` (default 40; raise for better recall at higher latency):

```sql
SET hnsw.ef_search = 100;
```

### Apply migrations

```bash
cd backend
alembic upgrade head
```

---

## Manual Setup

### Backend (Python / FastAPI)

```bash
cd backend
python -m venv .venv
source .venv/bin/activate        # Windows: .venv\Scripts\activate
pip install -r requirements.txt
```

Create a `.env` file — note that it contains **local endpoints and configuration only, no credentials at all**:

```env
# Ollama (local inference, no keys)
OLLAMA_BASE_URL=http://localhost:11434
OLLAMA_MODEL=llama3.2:3b
OLLAMA_EMBEDDING_MODEL=nomic-embed-text

# Reranker (local cross-encoder, runs in-process via sentence-transformers)
RERANK_MODEL=cross-encoder/ms-marco-MiniLM-L-6-v2

# Database (local Docker container)
POSTGRES_HOST=localhost
POSTGRES_DB=subtextai
POSTGRES_USER=subtextai
POSTGRES_PASSWORD=<local dev password>

# Redis
REDIS_URL=redis://localhost:6379/0

# Policies
MIN_WORD_COUNT=5
RERANK_SCORE_THRESHOLD_HIGH=0.70
RERANK_SCORE_THRESHOLD_LOW=0.45
RATE_LIMIT_PER_MINUTE=10

# Data protection
RAW_TEXT_RETENTION_DAYS=90

# Auth (self-issued JWT, no external identity provider)
JWT_SECRET=<local dev secret>
```

Run the backend:

```bash
uvicorn main:app --reload --port 8000
```

### Index documents

Place source documents in `scripts/docs/` (PDF or plain text) and run the indexing script, which handles chunking, vectorization, and storage:

```bash
cd scripts
python index_documents.py --source ./docs
```

The corpus is configurable: the script accepts any compatible collection without changes to the backend or governance logic.

> Re-indexing with a different embedding model invalidates the stored vectors **and** the calibration of the confidence thresholds. Run the evaluation harness afterwards — see [AI Pipeline](ai-pipeline.md).

### Frontend (React)

```bash
cd frontend
npm install
cp .env.example .env.local
```

Edit `.env.local`:

```env
VITE_API_BASE_URL=http://localhost:8000/api/v1
```

Run the frontend:

```bash
npm run dev                      # → http://localhost:5173
```

---

## Development

```bash
# Backend (from backend/)
uvicorn main:app --reload --port 8000     # → http://localhost:8000

# Frontend (from frontend/, separate terminal)
npm run dev                               # → http://localhost:5173
```

System status: `http://localhost:8000/api/v1/health`

### Running tests

```bash
# Backend tests
cd backend
pytest

# Frontend tests
cd frontend
npm run test
```

### Code quality

```bash
ruff check .
ruff format .

pre-commit install
pre-commit run --all-files
```

---

## Continuous Integration

The `.github/workflows/` directory contains pipelines for:
- Backend: lint (Ruff), test (Pytest)
- Frontend: lint, test, build
- Scheduled: evaluation harness run, publishing metrics to the local `/metrics` store

There is no deploy step and no cloud credential in CI — the project is designed to run entirely on the developer's own machine, not to be pushed to a hosting provider.

---

## Capacity Planning

### Local hardware is the real ceiling

There is no token quota and no per-minute billing limit — Ollama runs on your own GPU, so the constraint is hardware, not a cloud contract. The user-facing rate limit (10 req/min per user) still protects against runaway local load, but the practical ceiling is how many generations your GPU can serve concurrently.

On a single consumer GPU (e.g., 4GB VRAM), Ollama effectively serves **one generation at a time** — concurrent requests queue rather than run in parallel. This is very different from a cloud deployment's token-per-minute model and should shape expectations about how many simultaneous users the local setup can realistically support.

Mitigations, in order of impact:
1. Semantic caching (see [Roadmap](roadmap.md)) — avoids the analysis call entirely on equivalent messages
2. Trimming retrieved fragment count, which dominates prompt size and generation time
3. Using a smaller quantized model if latency matters more than analysis depth
4. Replacing the prompt-based crisis classifier with a lighter, fine-tuned local model

---

## Environment Variables

### Backend

| Variable | Description |
|-|-|
| `OLLAMA_BASE_URL` | Local Ollama server address (default: `http://localhost:11434`) |
| `OLLAMA_MODEL` | Generation model (default: `llama3.2:3b`) |
| `OLLAMA_EMBEDDING_MODEL` | Embedding model (default: `nomic-embed-text`) |
| `RERANK_MODEL` | Cross-encoder reranker identity — persisted in traces |
| `POSTGRES_HOST` / `POSTGRES_DB` / `POSTGRES_USER` / `POSTGRES_PASSWORD` | Local database connection |
| `REDIS_URL` | Redis connection string |
| `MIN_WORD_COUNT` | Minimum words per message (default: 5) |
| `RERANK_SCORE_THRESHOLD_HIGH` | HIGH confidence gate (default: 0.70) |
| `RERANK_SCORE_THRESHOLD_LOW` | Blocking threshold (default: 0.45) |
| `RATE_LIMIT_PER_MINUTE` | Request limit per minute per user (default: 10) |
| `RAW_TEXT_RETENTION_DAYS` | TTL before raw message text is purged (default: 90) |
| `JWT_SECRET` | Signing secret for self-issued JWTs |

No third-party API keys exist anywhere in this configuration — every value here points at something running on the local machine.

### Frontend

| Variable | Description |
|-|-|
| `VITE_API_BASE_URL` | Backend API base URL |
