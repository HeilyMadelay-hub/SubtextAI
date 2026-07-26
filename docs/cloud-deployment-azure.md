# Cloud Deployment (Azure) — Optional, Paid

Everything else in this repository describes the **default stack: fully local, fully free**, running on Ollama, Docker, and a local PostgreSQL instance (see [Deployment](deployment.md)). This document is different — it describes an **alternative, optional path**: what SubtextAI would look like deployed on Azure, with real cloud costs, in case the project ever needs to run for real users instead of on a single developer's machine.

Nothing here is currently implemented. This is a reference for a possible future migration, kept separate from the core docs so the local-first stack stays the single source of truth for how the project actually runs today.

---

## What Changes vs. the Local Stack

Only the AI layer, hosting, security, and observability change. The governance pipeline itself (policy validation → crisis/retrieval → rerank + gate → analysis → traceability) stays identical — that logic lives in the backend code, not in any specific cloud service.

| Layer | Local (current, free) | Azure (optional, paid) |
|-|-|-|
| **Generation model** | Ollama, `llama3.2:3b` | Azure OpenAI, GPT-4.1 |
| **Embedding model** | Ollama, `nomic-embed-text` (768-dim) | Azure AI Embeddings, `text-embedding-3-large` (3072-dim) |
| **Backend hosting** | Uvicorn, local machine | Azure Container Apps or Azure App Service |
| **Frontend hosting** | Vite dev server / local static build | Azure Static Web Apps |
| **Database** | PostgreSQL + pgvector, Docker container | Azure Database for PostgreSQL Flexible Server + pgvector |
| **Cache** | Redis, Docker container | Azure Cache for Redis |
| **Document storage** | Local filesystem | Azure Blob Storage |
| **Auth** | Self-issued JWT | Microsoft Entra ID (Azure AD) + OAuth2 + RBAC |
| **Service-to-service auth** | None needed (everything local) | Managed Identity via `DefaultAzureCredential` — no API keys |
| **Secrets** | None required | Azure Key Vault, read through Managed Identity |
| **Observability** | Local structured logs + OpenTelemetry console exporter | Azure Monitor + Application Insights |
| **CI/CD** | GitHub Actions (lint + test only) | GitHub Actions with OIDC federated credentials, deploying to Azure |

> **Embedding dimension note.** Switching from `nomic-embed-text` (768-dim) to `text-embedding-3-large` (3072-dim) means re-indexing the entire corpus and updating the `vector(768)` column in `document_chunks` to `vector(3072)`. It also invalidates the calibration of the confidence thresholds — the evaluation harness must be re-run against the new embedding model before trusting the `0.70` / `0.45` cutoffs again (see [AI Pipeline](ai-pipeline.md)).

---

## Authentication: Managed Identity

In the local stack there is nothing to authenticate — everything runs on one machine. On Azure, the equivalent principle is **no standing credentials**: the backend never stores an API key. All service-to-service authentication goes through **Microsoft Entra ID** via `DefaultAzureCredential`, which resolves to a Managed Identity in Azure (or the developer's `az login` session locally, if developing against live Azure services).

### Why no keys

API keys in configuration are a standing credential: they leak through logs, backups, and environment dumps, and rotating them requires a redeploy. Managed Identity issues short-lived tokens automatically, scoped by RBAC, with nothing to rotate and nothing to leak.

### Backend client setup

```python
from azure.identity.aio import DefaultAzureCredential, get_bearer_token_provider
from openai import AsyncAzureOpenAI

credential = DefaultAzureCredential()
token_provider = get_bearer_token_provider(
    credential, "https://cognitiveservices.azure.com/.default"
)

client = AsyncAzureOpenAI(
    azure_endpoint=settings.azure_openai_endpoint,
    azure_ad_token_provider=token_provider,
    api_version=settings.azure_openai_api_version,
)
```

### Enable the identity and assign roles

```bash
# Enable system-assigned managed identity on the backend
az webapp identity assign \
  --resource-group subtextai-rg \
  --name subtextai-api

PRINCIPAL_ID=$(az webapp identity show \
  --resource-group subtextai-rg \
  --name subtextai-api \
  --query principalId -o tsv)

# Azure OpenAI access
az role assignment create \
  --assignee "$PRINCIPAL_ID" \
  --role "Cognitive Services OpenAI User" \
  --scope "/subscriptions/<sub-id>/resourceGroups/subtextai-rg/providers/Microsoft.CognitiveServices/accounts/<openai-account>"

# Key Vault access (for the few secrets that are not Azure-native)
az role assignment create \
  --assignee "$PRINCIPAL_ID" \
  --role "Key Vault Secrets User" \
  --scope "/subscriptions/<sub-id>/resourceGroups/subtextai-rg/providers/Microsoft.KeyVault/vaults/<vault-name>"

# Blob Storage access (RAG corpus)
az role assignment create \
  --assignee "$PRINCIPAL_ID" \
  --role "Storage Blob Data Reader" \
  --scope "/subscriptions/<sub-id>/resourceGroups/subtextai-rg/providers/Microsoft.Storage/storageAccounts/<storage-account>"
```

### PostgreSQL with Entra ID

Azure Database for PostgreSQL Flexible Server supports Entra ID authentication, so the database connection also avoids a stored password:

```bash
az postgres flexible-server ad-admin create \
  --resource-group subtextai-rg \
  --server-name subtextai-db \
  --display-name subtextai-api \
  --object-id "$PRINCIPAL_ID"
```

The backend requests a token for `https://ossrdbms-aad.database.windows.net/.default` and uses it as the connection password.

---

## Why pgvector Rather Than a Managed Vector Search Service

Managed vector search services (e.g., Azure AI Search) offer hybrid retrieval and a semantic reranker as built-in features. pgvector does not — it provides vector similarity only, and the hybrid layer has to be built explicitly (see [AI Pipeline](ai-pipeline.md) for the SQL and fusion logic).

pgvector was chosen for the default local stack anyway, because it removes an entire service from the topology: traces, vectors, and audit records live in one PostgreSQL instance with one consistency model and one backup story, and it needs no external account or network dependency at all. The cost is roughly 200 lines of retrieval code, which is a fair trade at this scale — and the only option that keeps the project at zero cost and fully offline.

A managed service like Azure AI Search becomes the better option once corpus size or query volume outgrows what a single local PostgreSQL instance handles comfortably — trading that 200 lines of retrieval code for a managed feature, at the cost of an ongoing bill and an external dependency.

---

## Database Setup on Flexible Server

Extensions must be allowlisted at the server level before they can be created — unlike a local Docker instance, this is not enabled by default:

```bash
az postgres flexible-server parameter set \
  --resource-group subtextai-rg \
  --server-name subtextai-db \
  --name azure.extensions \
  --value vector
```

Then, inside the database:

```sql
CREATE EXTENSION IF NOT EXISTS vector;
```

The same HNSW and GIN indexes described in [AI Pipeline](ai-pipeline.md) apply unchanged.

---

## Backend Deployment

### Azure Container Apps (default choice)

Container Apps scales to zero between bursts, which suits a workload whose cost is dominated by per-request model calls rather than steady compute.

```bash
docker build -f docker/Dockerfile.backend -t subtextai-api .
az acr login --name <your-registry>
docker tag subtextai-api <your-registry>.azurecr.io/subtextai-api
docker push <your-registry>.azurecr.io/subtextai-api

az containerapp create \
  --name subtextai-api \
  --resource-group subtextai-rg \
  --image <your-registry>.azurecr.io/subtextai-api \
  --target-port 8000 \
  --ingress external \
  --system-assigned \
  --min-replicas 0 \
  --max-replicas 10
```

**Cold start caveat.** Scaling to zero adds several seconds to the first request after an idle period. Set `--min-replicas 1` if consistent latency matters more than idle cost.

### Alternative: Azure App Service

Choose App Service for a predictable, always-on baseline without cold starts:

```bash
az webapp create \
  --resource-group subtextai-rg \
  --plan subtextai-plan \
  --name subtextai-api \
  --deployment-container-image-name <your-registry>.azurecr.io/subtextai-api
```

### Frontend → Azure Static Web Apps

```bash
cd frontend
npm run build
az staticwebapp deploy --app-location "." --output-location "dist" --name subtextai-frontend
```

Update `VITE_API_BASE_URL` in the Static Web App configuration to point to the production backend URL.

### CI/CD with GitHub Actions

The `.github/workflows/` directory would add pipelines for:
- Backend: lint (Ruff), test (Pytest), build image, deploy to Azure
- Frontend: lint, test, build, deploy to Static Web Apps
- Scheduled: evaluation harness run, publishing metrics to Application Insights

Workflows authenticate to Azure with **OIDC federated credentials**, so no service-principal secret is stored in GitHub.

---

## Capacity Planning: Token Quota Is the Real Ceiling

This is the biggest practical difference from the local stack. Locally, the ceiling is GPU throughput (see [Deployment](deployment.md#capacity-planning)) — one generation at a time, no billing. On Azure OpenAI, the ceiling is the deployment's **TPM (tokens per minute)** quota, and it costs money per token regardless of concurrency.

Each analysis consumes two calls against that quota:

| Call | Approximate tokens |
|-|-|
| Crisis classifier | ~200 in / ~20 out |
| Pragmatic analysis | ~2,000 in (prompt + retrieved fragments) / ~400 out |
| **Per request** | **~2,600 tokens** |

At a default 30K TPM deployment that is roughly **11 requests/minute sustained**, regardless of how many users are connected. Plan quota against expected concurrency, not against the per-user rate limit.

Mitigations, in order of impact:
1. Semantic caching (see [Roadmap](roadmap.md)) — avoids the analysis call entirely on equivalent messages
2. Provisioned Throughput Units for predictable latency at sustained load
3. Trimming retrieved fragment count, which dominates input tokens
4. Replacing the prompt-based crisis classifier with a fine-tuned model

Monitor `AzureOpenAI/TokenTransaction` and 429 rates in Application Insights.

---

## Observability

| Layer | Technology |
|-|-|
| **Distributed tracing** | OpenTelemetry, exported to Application Insights |
| **Metrics & alerts** | Application Insights + Azure Monitor |
| **Logging** | Structured logging, shipped to Log Analytics |
| **Health checks** | `/health` endpoint monitoring all connected Azure services |

Azure Monitor receives aggregated metrics from both production interactions and the evaluation job, enabling behavior monitoring and quantitative prompt version comparison at a scale a local console exporter can't provide.

---

## Data Residency and Compliance Notes

The [GDPR-aware storage design](design-principles.md#traceability-under-data-protection) (split storage, text-level erasure, retention TTL) does not change on Azure — it is implemented in the backend, independent of where it runs. What does change:

- **Region.** All resources should be deployed in a single, explicitly configured region matching where users are located.
- **Data residency.** Azure OpenAI must be provisioned in an EU region for EU users if that's a requirement.
- **No training.** Azure OpenAI does not use API traffic to train models; this guarantee is specific to Azure OpenAI's terms and would need re-verification for any other cloud provider.

---

## Environment Variables (Azure)

| Variable | Description |
|-|-|
| `AZURE_OPENAI_ENDPOINT` | Azure OpenAI endpoint |
| `AZURE_OPENAI_DEPLOYMENT` | Deployment name (`gpt-4.1`) |
| `AZURE_OPENAI_API_VERSION` | API version |
| `AZURE_EMBEDDINGS_DEPLOYMENT` | Embeddings model deployment (`text-embedding-3-large`) |
| `RERANK_MODEL` / `RERANK_ENDPOINT` | Reranker identity and endpoint, if also hosted on Azure |
| `POSTGRES_HOST` / `POSTGRES_DB` / `POSTGRES_USER` | Flexible Server connection (token-authenticated, no password) |
| `AZURE_AD_TENANT_ID` / `AZURE_AD_CLIENT_ID` | Entra ID tenant and application client ID |

Policy thresholds (`RERANK_SCORE_THRESHOLD_HIGH`, `RERANK_SCORE_THRESHOLD_LOW`, `RATE_LIMIT_PER_MINUTE`, `RAW_TEXT_RETENTION_DAYS`, `MIN_WORD_COUNT`) are unchanged from the local `.env` — those are pipeline behavior, not infrastructure.

---

## When This Path Would Actually Make Sense

- The corpus or query volume outgrows what a single local PostgreSQL instance and a 4GB-VRAM GPU can serve comfortably.
- The project needs to be reachable by users other than the developer — a real deployment, not a local demo.
- Latency needs to drop from the ~8-14s per LLM call measured locally (see [AI Pipeline](ai-pipeline.md)) down to the ~1.6-2.3s range GPT-4.1 provides on provisioned cloud compute.

Until one of those becomes true, the local stack remains the right default: zero cost, zero external dependency, and no pressure to keep a cloud bill under control while the project is still being built.
