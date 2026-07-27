# Cloud Deployment (AWS) — Production

This is how SubtextAI runs in production: on AWS, with generation and embeddings served through **OpenRouter**. There is no local-only mode anymore — the stack described here is the real, current deployment target, not a hypothetical future migration.

Only the AI layer, hosting, security, and observability are cloud-specific. The governance pipeline itself (policy validation → crisis/retrieval → rerank + gate → analysis → traceability) is unchanged — that logic lives in the backend code, not in any specific cloud service.

---

## Production Topology

| Layer | Service | Why this one |
|-|-|-|
| **Generation model** | OpenRouter, `openai/gpt-4.1` | Access to GPT-4.1 (and any other frontier model) through a single API and a single billing relationship, without a direct Azure or OpenAI account. Swapping the underlying model later is a config change, not a rewrite. |
| **Embedding model** | OpenRouter, `text-embedding-3-large` (3072-dim) | Keeps the embedding call on the same provider and billing surface as generation — one API key, one rate-limit budget, one place to observe cost instead of two. |
| **Backend hosting** | AWS App Runner | Scales to zero between bursts and needs no cluster to operate, which fits a workload whose cost is dominated by per-request model calls rather than steady compute. |
| **Frontend hosting** | AWS Amplify Hosting | Git-connected static hosting with CI built in — push to the tracked branch and it builds and deploys the Vite output, no separate pipeline needed for the frontend. |
| **Database** | Amazon RDS for PostgreSQL + pgvector | A managed, single-engine home for traces, vectors, and audit records, with automated backups and patching — same reasoning that justified pgvector locally (one consistency model, one service) now applied to a managed instance. |
| **Cache** | Amazon ElastiCache for Redis | Drop-in managed replacement for the Redis container — same client, same data structures, no application code change. |
| **Document storage** | Amazon S3 | Durable, versioned storage for the RAG corpus, with lifecycle rules for old document versions and native integration with IAM for access control. |
| **Auth** | Amazon Cognito + OAuth2 + IAM | Offloads user identity, token issuance, and MFA to a managed identity provider instead of a self-issued JWT the backend has to secure and rotate by hand. |
| **Service-to-service auth** | IAM roles (App Runner / ECS task role) | Short-lived, automatically rotated credentials scoped by policy for every AWS-native call (RDS, S3, Secrets Manager, CloudWatch) — nothing to leak through logs or environment dumps. |
| **Secrets** | AWS Secrets Manager | The one place that holds the OpenRouter API key (the one credential that IAM cannot cover, since OpenRouter is outside the AWS trust boundary) plus any non-IAM database fallback credentials. |
| **Observability** | Amazon CloudWatch + AWS X-Ray + OpenTelemetry | Traces, metrics, and logs in the same account as the workload, with X-Ray giving request-level spans across App Runner, RDS, and outbound OpenRouter calls. |
| **CI/CD** | GitHub Actions with OIDC federated to AWS IAM, deploying to App Runner / Amplify | No long-lived AWS access key sits in GitHub — the workflow assumes a role for the duration of the run and nothing else. |

> **Embedding dimension note.** The corpus is indexed against `text-embedding-3-large` (3072-dim), so the `document_chunks.embedding` column is `vector(3072)`, not `vector(768)`. Changing the embedding model again means re-indexing the entire corpus and re-running the evaluation harness before trusting the `0.70` / `0.45` confidence cutoffs — see [AI Pipeline](ai-pipeline.md).

---

## Authentication: IAM Roles, and the One Exception

For every AWS-native service — RDS, S3, Secrets Manager, CloudWatch — the backend authenticates with **no standing credential**. App Runner (or the ECS task) assumes an IAM role, and the AWS SDK resolves short-lived, automatically rotated credentials from the instance metadata service. Nothing is stored in configuration, nothing needs to be rotated by hand, and nothing can leak through a log or an environment dump.

### Why OpenRouter is different

OpenRouter is not an AWS service — it sits outside the IAM trust boundary, so there is no role to assume against it. Authenticating to it necessarily means a standing API key. The design goal, then, isn't "no key" (impossible for a third party) but **the key never touches configuration, source, or a log**: it lives in Secrets Manager and is resolved by the backend at startup using its own IAM role, which *is* keyless.

```python
import boto3
import json
from openai import AsyncOpenAI

def get_openrouter_key() -> str:
    client = boto3.client("secretsmanager", region_name=settings.aws_region)
    secret = client.get_secret_value(SecretId="subtextai/openrouter-api-key")
    return json.loads(secret["SecretString"])["api_key"]

client = AsyncOpenAI(
    base_url="https://openrouter.ai/api/v1",
    api_key=get_openrouter_key(),
)
```

### Enable the IAM role and attach policies

```bash
# Create the App Runner instance role (assumed by the running service)
aws iam create-role \
  --role-name subtextai-api-role \
  --assume-role-policy-document file://apprunner-trust-policy.json

# Secrets Manager access (OpenRouter key only — scoped to a single secret ARN)
aws iam put-role-policy \
  --role-name subtextai-api-role \
  --policy-name subtextai-secrets-access \
  --policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Action": "secretsmanager:GetSecretValue",
      "Resource": "arn:aws:secretsmanager:<region>:<account-id>:secret:subtextai/openrouter-api-key-*"
    }]
  }'

# S3 access (RAG corpus)
aws iam put-role-policy \
  --role-name subtextai-api-role \
  --policy-name subtextai-s3-access \
  --policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:ListBucket"],
      "Resource": ["arn:aws:s3:::subtextai-corpus", "arn:aws:s3:::subtextai-corpus/*"]
    }]
  }'
```

### PostgreSQL with IAM Authentication

RDS for PostgreSQL supports IAM database authentication, so the database connection avoids a stored password as well:

```bash
aws rds modify-db-instance \
  --db-instance-identifier subtextai-db \
  --enable-iam-database-authentication

aws rds-db generate-db-auth-token \
  --hostname subtextai-db.xxxxxxxx.<region>.rds.amazonaws.com \
  --port 5432 \
  --username subtextai_api \
  --region <region>
```

The backend requests a signed auth token via the same IAM role and uses it as the connection password — it rotates automatically, with nothing to leak.

---

## Why pgvector Rather Than a Managed Vector Search Service

Managed vector search services (e.g., Amazon OpenSearch with the k-NN plugin) offer hybrid retrieval and reranking as built-in features. pgvector does not — it provides vector similarity only, and the hybrid layer has to be built explicitly (see [AI Pipeline](ai-pipeline.md) for the SQL and fusion logic).

pgvector was chosen anyway, because it removes an entire service from the topology: traces, vectors, and audit records live in one RDS instance with one consistency model and one backup story. The cost is roughly 200 lines of retrieval code, which is a fair trade at this scale.

A managed service like OpenSearch becomes the better option once corpus size or query volume outgrows what a single RDS instance handles comfortably — trading that 200 lines of retrieval code for a managed feature, at the cost of an additional service and an additional bill.

---

## Database Setup on RDS

The `vector` extension must be added to the instance's parameter group before it can be created — unlike a local Docker instance, it is not enabled by default:

```bash
aws rds modify-db-parameter-group \
  --db-parameter-group-name subtextai-pg \
  --parameters "ParameterName=shared_preload_libraries,ParameterValue=vector,ApplyMethod=pending-reboot"
```

Then, inside the database:

```sql
CREATE EXTENSION IF NOT EXISTS vector;
```

The same HNSW and GIN indexes described in [AI Pipeline](ai-pipeline.md) apply unchanged.

---

## Backend Deployment

### AWS App Runner (default choice)

```bash
docker build -f docker/Dockerfile.backend -t subtextai-api .

aws ecr get-login-password --region <region> | docker login --username AWS --password-stdin <account-id>.dkr.ecr.<region>.amazonaws.com
docker tag subtextai-api <account-id>.dkr.ecr.<region>.amazonaws.com/subtextai-api
docker push <account-id>.dkr.ecr.<region>.amazonaws.com/subtextai-api

aws apprunner create-service \
  --service-name subtextai-api \
  --source-configuration '{
    "ImageRepository": {
      "ImageIdentifier": "<account-id>.dkr.ecr.<region>.amazonaws.com/subtextai-api:latest",
      "ImageRepositoryType": "ECR",
      "ImageConfiguration": { "Port": "8000" }
    },
    "AutoDeploymentsEnabled": true
  }' \
  --instance-configuration '{ "InstanceRoleArn": "arn:aws:iam::<account-id>:role/subtextai-api-role" }'
```

**Cold start caveat.** App Runner's minimum instance count of 1 keeps a warm instance running at all times, which avoids the cold-start latency of a true scale-to-zero service — at the cost of a small always-on baseline charge. Set the minimum size to 0-capable only if idle cost matters more than consistent first-request latency.

### Alternative: Amazon ECS Fargate

Choose ECS Fargate for finer-grained control over networking, scaling policies, and multi-container task definitions:

```bash
aws ecs create-service \
  --cluster subtextai-cluster \
  --service-name subtextai-api \
  --task-definition subtextai-api-task \
  --desired-count 1 \
  --launch-type FARGATE \
  --network-configuration "awsvpcConfiguration={subnets=[<subnet-id>],securityGroups=[<sg-id>],assignPublicIp=ENABLED}"
```

### Frontend → AWS Amplify Hosting

```bash
cd frontend
npm run build

aws amplify create-app --name subtextai-frontend --repository <github-repo-url>
aws amplify create-branch --app-id <app-id> --branch-name main
aws amplify start-deployment --app-id <app-id> --branch-name main
```

Update `VITE_API_BASE_URL` in the Amplify app's environment variables to point to the production App Runner URL.

### CI/CD with GitHub Actions

The `.github/workflows/` directory adds pipelines for:
- Backend: lint (Ruff), test (Pytest), build image, push to ECR, deploy to App Runner
- Frontend: lint, test, build, deploy to Amplify
- Scheduled: evaluation harness run, publishing metrics to CloudWatch

Workflows authenticate to AWS with **OIDC federated credentials** via `aws-actions/configure-aws-credentials`, so no long-lived IAM access key is stored in GitHub.

---

## Capacity Planning: OpenRouter Rate Limits Are the Real Ceiling

This is the biggest practical difference from a self-hosted model. The ceiling is no longer GPU throughput — it is OpenRouter's per-key rate limit and, per token, its per-request cost.

Each analysis consumes two model calls against that limit:

| Call | Approximate tokens |
|-|-|
| Crisis classifier | ~200 in / ~20 out |
| Pragmatic analysis | ~2,000 in (prompt + retrieved fragments) / ~400 out |
| **Per request** | **~2,600 tokens** |

Plan the OpenRouter key's rate limit and monthly budget against expected concurrency, not against the per-user rate limit enforced by the backend's own policy engine.

Mitigations, in order of impact:
1. Semantic caching (see [Roadmap](roadmap.md)) — avoids the analysis call entirely on equivalent messages
2. Trimming retrieved fragment count, which dominates input tokens
3. Routing the crisis classifier to a smaller, cheaper model on OpenRouter while keeping GPT-4.1 for the main analysis
4. Replacing the prompt-based crisis classifier with a fine-tuned model

Monitor OpenRouter's usage dashboard and the `429` rate surfaced in CloudWatch.

---

## Observability

| Layer | Technology |
|-|-|
| **Distributed tracing** | OpenTelemetry, exported to AWS X-Ray |
| **Metrics & alerts** | Amazon CloudWatch |
| **Logging** | Structured logging, shipped to CloudWatch Logs |
| **Health checks** | `/health` endpoint monitoring all connected AWS services and the OpenRouter API |

CloudWatch receives aggregated metrics from both production interactions and the evaluation job, enabling behavior monitoring and quantitative prompt version comparison at a scale a local console exporter never could.

---

## Data Residency and Compliance Notes

The [GDPR-aware storage design](design-principles.md#traceability-under-data-protection) (split storage, text-level erasure, retention TTL) does not change on AWS — it is implemented in the backend, independent of where it runs. What does change:

- **Region.** All AWS resources should be deployed in a single, explicitly configured region matching where users are located.
- **OpenRouter is a third party.** Unlike a same-tenant Azure OpenAI deployment, OpenRouter is an external router that forwards requests to whichever underlying provider serves the selected model. Data residency and no-training guarantees are only as strong as OpenRouter's terms **and** the terms of the specific model it routes to — this needs explicit verification per model, not an assumption inherited from AWS's own compliance posture.
- **No training, verify per model.** Some model providers reachable through OpenRouter commit to not training on API traffic; others don't, or the commitment depends on a paid tier. This must be checked for the exact model in use before processing conversations that qualify as special-category data under GDPR.

---

## Environment Variables (AWS)

| Variable | Description |
|-|-|
| `OPENROUTER_API_KEY` | Resolved from Secrets Manager at startup, never set directly in the environment |
| `OPENROUTER_MODEL` | Generation model route (`openai/gpt-4.1`) |
| `OPENROUTER_EMBEDDING_MODEL` | Embedding model route (`text-embedding-3-large`) |
| `RERANK_MODEL` / `RERANK_ENDPOINT` | Reranker identity and endpoint, if also hosted on AWS |
| `POSTGRES_HOST` / `POSTGRES_DB` / `POSTGRES_USER` | RDS connection (IAM-token-authenticated, no password) |
| `AWS_REGION` | Region for all AWS SDK calls (Secrets Manager, S3, CloudWatch, RDS IAM auth) |
| `COGNITO_USER_POOL_ID` / `COGNITO_CLIENT_ID` | Cognito user pool and app client for OAuth2 token validation |

Policy thresholds (`RERANK_SCORE_THRESHOLD_HIGH`, `RERANK_SCORE_THRESHOLD_LOW`, `RATE_LIMIT_PER_MINUTE`, `RAW_TEXT_RETENTION_DAYS`, `MIN_WORD_COUNT`) are unchanged — those are pipeline behavior, not infrastructure.
