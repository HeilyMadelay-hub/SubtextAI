# Deployment Guide

This covers getting the application itself running against its AWS resources — environment configuration, migrations, and the dev loop. For provisioning the AWS resources themselves (the EC2 instance, IAM instance profile, S3, Secrets Manager) and the reasoning behind each service, see [Cloud Deployment (AWS)](cloud-deployment-aws.md).

## Prerequisites

- Python 3.13+
- Node.js 18+ and npm
- Docker and Docker Compose (for building the backend image and running Postgres/Redis alongside it)
- An AWS account with the AWS CLI configured, an EC2 key pair, and access to the provisioned S3 bucket
- An [OpenRouter](https://openrouter.ai) API key, stored in AWS Secrets Manager (see [Cloud Deployment](cloud-deployment-aws.md#authentication-iam-instance-profile-and-the-one-exception))

---

## Clone the repository

```bash
git clone https://github.com/<your-user>/subtextai.git
cd subtextai
```

---

## Generation and Embeddings via OpenRouter

**The application makes no direct calls to a model provider.** Every generation and embedding call goes through OpenRouter with a single API key, resolved from Secrets Manager at startup — never stored in `.env` or source.

```python
import boto3, json
from openai import AsyncOpenAI

def get_openrouter_key() -> str:
    client = boto3.client("secretsmanager", region_name=settings.aws_region)
    secret = client.get_secret_value(SecretId="subtextai/openrouter-api-key")
    return json.loads(secret["SecretString"])["api_key"]

client = AsyncOpenAI(
    base_url="https://openrouter.ai/api/v1",
    api_key=get_openrouter_key(),
)

response = await client.chat.completions.create(
    model="openai/gpt-4.1",
    messages=[{"role": "user", "content": prompt}],
    response_format={"type": "json_object"},
)
```

---

## Database Setup

### Enable the pgvector extension

Postgres runs as a Docker container (`pgvector/pgvector:pg16`) on the EC2 instance, which already ships the extension — no parameter group to configure, unlike a managed instance (see [Cloud Deployment](cloud-deployment-aws.md#database-setup-on-docker)). Just create it inside the database:

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

## Application Setup

### Backend (Python / FastAPI)

```bash
cd backend
python -m venv .venv
source .venv/bin/activate        # Windows: .venv\Scripts\activate
pip install -r requirements.txt
```

Create a `.env` file — note that it contains **AWS endpoints and resource identifiers, not credentials**; every credential is resolved at runtime via IAM or Secrets Manager:

```env
# OpenRouter (key resolved from Secrets Manager, not set here)
OPENROUTER_SECRET_ID=subtextai/openrouter-api-key
OPENROUTER_MODEL=openai/gpt-4.1
OPENROUTER_EMBEDDING_MODEL=text-embedding-3-large

# Reranker (cross-encoder, runs in-process via sentence-transformers)
RERANK_MODEL=cross-encoder/ms-marco-MiniLM-L-6-v2

# Database (PostgreSQL, Docker container on the same EC2 instance)
POSTGRES_HOST=postgres          # Docker Compose service name — internal network only
POSTGRES_DB=subtextai
POSTGRES_USER=subtextai_api

# Cache (Redis, Docker container on the same EC2 instance)
REDIS_URL=redis://redis:6379/0  # Docker Compose service name — internal network only

# AWS
AWS_REGION=<region>

# Auth (Amazon Cognito)
COGNITO_USER_POOL_ID=<user-pool-id>
COGNITO_CLIENT_ID=<app-client-id>

# Policies
MIN_WORD_COUNT=5
RERANK_SCORE_THRESHOLD_HIGH=0.70
RERANK_SCORE_THRESHOLD_LOW=0.45
RATE_LIMIT_PER_MINUTE=10

# Data protection
RAW_TEXT_RETENTION_DAYS=90
```

Run the backend:

```bash
uvicorn main:app --reload --port 8000
```

### Index documents

Documents live in an S3 bucket (see [Cloud Deployment](cloud-deployment-aws.md)). The indexing script pulls from S3, and handles chunking, vectorization, and storage:

```bash
cd scripts
python index_documents.py --source s3://subtextai-corpus
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
VITE_API_BASE_URL=https://<ec2-instance-domain>/api/v1
```

Run the frontend against the deployed backend:

```bash
npm run dev                      # → http://localhost:5173
```

---

## Development Loop

```bash
# Backend (from backend/)
uvicorn main:app --reload --port 8000     # → http://localhost:8000

# Frontend (from frontend/, separate terminal)
npm run dev                               # → http://localhost:5173
```

The dev server talks to real AWS resources and to Postgres/Redis containers (either run locally with `docker compose up postgres redis` for a dev loop, or against the production EC2 containers over an SSH tunnel) — there is no offline/local-only mode. System status: `http://localhost:8000/api/v1/health`.

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

## Continuous Integration and Deployment

The `.github/workflows/` directory contains pipelines for:
- Backend: lint (Ruff), test (Pytest), build image, push to ECR, SSH into the EC2 instance to pull the new image and `docker compose up -d`
- Frontend: lint, test, build, deploy to Amplify
- Scheduled: evaluation harness run, publishing metrics to CloudWatch

Workflows authenticate to AWS via **OIDC federated credentials** — no long-lived IAM access key is stored in GitHub. See [Cloud Deployment](cloud-deployment-aws.md#cicd-with-github-actions) for the full setup.

---

## Capacity Planning

See [Cloud Deployment — Capacity Planning](cloud-deployment-aws.md#capacity-planning-two-ceilings-now-not-one): OpenRouter's rate limit and per-token cost is one constraint, and `t3.micro`'s CPU credits and 1GB RAM — now shared across FastAPI, Postgres, Redis, and Nginx on the same instance — is the other.

---

## Environment Variables

### Backend

| Variable | Description |
|-|-|
| `OPENROUTER_SECRET_ID` | Secrets Manager secret ID holding the OpenRouter API key |
| `OPENROUTER_MODEL` | Generation model route (default: `openai/gpt-4.1`) |
| `OPENROUTER_EMBEDDING_MODEL` | Embedding model route (default: `text-embedding-3-large`) |
| `RERANK_MODEL` | Cross-encoder reranker identity — persisted in traces |
| `POSTGRES_HOST` / `POSTGRES_DB` / `POSTGRES_USER` | Connection to the local Postgres container (Docker Compose service name, internal network only) |
| `REDIS_URL` | Connection to the local Redis container (Docker Compose service name, internal network only) |
| `AWS_REGION` | Region for all AWS SDK calls |
| `COGNITO_USER_POOL_ID` / `COGNITO_CLIENT_ID` | Cognito user pool and app client for token validation |
| `MIN_WORD_COUNT` | Minimum words per message (default: 5) |
| `RERANK_SCORE_THRESHOLD_HIGH` | HIGH confidence gate (default: 0.70) |
| `RERANK_SCORE_THRESHOLD_LOW` | Blocking threshold (default: 0.45) |
| `RATE_LIMIT_PER_MINUTE` | Request limit per minute per user (default: 10) |
| `RAW_TEXT_RETENTION_DAYS` | TTL before raw message text is purged (default: 90) |

The only secret value in this list is the OpenRouter API key, and it is never set directly — it is resolved from Secrets Manager via the backend's IAM role.

### Frontend

| Variable | Description |
|-|-|
| `VITE_API_BASE_URL` | Backend API base URL (EC2 instance domain, through Nginx) |
