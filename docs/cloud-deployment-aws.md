# Cloud Deployment (AWS) — Production

This is how SubtextAI runs in production: on AWS, sized to run entirely within the **AWS Free Tier**, with generation and embeddings served through **OpenRouter**. There is no local-only mode anymore — the stack described here is the real, current deployment target, not a hypothetical future migration.

The governance pipeline itself (policy validation → crisis/retrieval → rerank + gate → analysis → traceability) is unchanged regardless of hosting cost — that logic lives in the backend code, not in any specific cloud service. What changes here is how cheaply that code can be run for a single-developer, portfolio-scale deployment without giving up production practices: containerization, IAM-scoped access, managed identity, and centralized observability all stay.

---

## Production Topology

| Layer | Service | Why this one |
|-|-|-|
| **Generation model** | OpenRouter, `openai/gpt-4.1` | Access to GPT-4.1 (and any other frontier model) through a single API and a single billing relationship, without a direct provider account. Swapping the underlying model later is a config change, not a rewrite. |
| **Embedding model** | OpenRouter, `text-embedding-3-large` (3072-dim) | Keeps the embedding call on the same provider and billing surface as generation — one API key, one rate-limit budget, one place to observe cost instead of two. |
| **Backend hosting** | Amazon EC2 `t3.micro` (Docker) | The Free Tier covers 750 hours/month of `t3.micro` — enough for one instance running continuously at no compute cost. A managed container service (App Runner, ECS Fargate) bills per vCPU-second with no free tier at this scale, which is the wrong trade for a portfolio-stage project. |
| **Reverse proxy** | Nginx (on the same EC2 instance) | Terminates HTTP(S), handles the public-facing port, and proxies to the FastAPI container on its internal port — the same role a managed load balancer would play, at zero additional cost. |
| **Database** | PostgreSQL + pgvector, in a Docker container on the EC2 instance | RDS has a Free Tier allowance too, but running Postgres in its own container on the same box removes a second billed resource and a second network hop, at the cost of managing backups yourself instead of getting them automatically. |
| **Cache** | Redis, in a Docker container on the EC2 instance | Same reasoning as the database: ElastiCache has no meaningful free tier for a persistent cache node, and Redis is light enough to share the instance with the rest of the stack. |
| **Frontend hosting** | AWS Amplify Hosting | Free Tier includes build minutes and hosting bandwidth generous enough for a low-traffic frontend — no change from a fully managed deployment. |
| **Document storage** | Amazon S3 | Free Tier covers 5GB, well above what a document corpus for this project needs. |
| **Auth** | Amazon Cognito + OAuth2 + IAM | Cognito's free tier covers the user volumes a portfolio project will see — no reason to self-issue JWTs and take on that surface area. |
| **Service-to-service auth** | IAM instance profile (EC2) | The EC2 instance assumes a role the same way an App Runner or ECS task would — short-lived, automatically rotated credentials for every AWS-native call (S3, Secrets Manager, CloudWatch), with nothing to leak through logs or environment dumps. |
| **Secrets** | AWS Secrets Manager | Holds the OpenRouter API key (the one credential IAM cannot cover, since OpenRouter is outside the AWS trust boundary). Secrets Manager has a small monthly per-secret cost outside its 30-day trial — the one line item in this stack that isn't strictly free, and worth it for keeping the key out of configuration. |
| **Observability** | Amazon CloudWatch + AWS X-Ray + OpenTelemetry | Free Tier includes a baseline of CloudWatch metrics/logs and X-Ray traces — enough for a single-instance workload without hitting billable volume. |
| **CI/CD** | GitHub Actions with OIDC federated to AWS IAM, deploying to EC2 / Amplify | No long-lived AWS access key sits in GitHub — the workflow assumes a role for the duration of the run and nothing else. |

> **Embedding dimension note.** The corpus is indexed against `text-embedding-3-large` (3072-dim), so the `document_chunks.embedding` column is `vector(3072)`. Changing the embedding model again means re-indexing the entire corpus and re-running the evaluation harness before trusting the `0.70` / `0.45` confidence cutoffs — see [AI Pipeline](ai-pipeline.md).

> **Why not managed services outright.** RDS, ElastiCache, App Runner, and ECS Fargate remain the better choice once the project outgrows a single instance — they trade a monthly bill for automated backups, patching, and horizontal scaling. For a single-developer deployment with modest traffic, that trade isn't worth it yet; the architecture is deliberately structured so that migrating to them later is a hosting change, not a rewrite of the governance pipeline.

---

## Authentication: IAM Instance Profile, and the One Exception

For every AWS-native service — S3, Secrets Manager, CloudWatch — the backend authenticates with **no standing credential**. The EC2 instance is launched with an **IAM instance profile**, and the AWS SDK resolves short-lived, automatically rotated credentials from the instance metadata service (IMDSv2). Nothing is stored in configuration, nothing needs to be rotated by hand, and nothing can leak through a log or an environment dump — the same guarantee a managed compute service like App Runner or ECS would give, just attached to a plain EC2 instance instead.

### Why OpenRouter is different

OpenRouter is not an AWS service — it sits outside the IAM trust boundary, so there is no role to assume against it. Authenticating to it necessarily means a standing API key. The design goal, then, isn't "no key" (impossible for a third party) but **the key never touches configuration, source, or a log**: it lives in Secrets Manager and is resolved by the backend at startup using the instance's own IAM role, which *is* keyless.

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

### Create the instance profile and attach policies

```bash
# Trust policy: EC2 is allowed to assume this role
aws iam create-role \
  --role-name subtextai-ec2-role \
  --assume-role-policy-document file://ec2-trust-policy.json

# Secrets Manager access (OpenRouter key only — scoped to a single secret ARN)
aws iam put-role-policy \
  --role-name subtextai-ec2-role \
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
  --role-name subtextai-ec2-role \
  --policy-name subtextai-s3-access \
  --policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:ListBucket"],
      "Resource": ["arn:aws:s3:::subtextai-corpus", "arn:aws:s3:::subtextai-corpus/*"]
    }]
  }'

# CloudWatch (logs + metrics + X-Ray)
aws iam attach-role-policy \
  --role-name subtextai-ec2-role \
  --policy-arn arn:aws:iam::aws:policy/CloudWatchAgentServerPolicy

aws iam create-instance-profile --instance-profile-name subtextai-ec2-profile
aws iam add-role-to-instance-profile \
  --instance-profile-name subtextai-ec2-profile \
  --role-name subtextai-ec2-role
```

The EC2 instance is launched with `--iam-instance-profile Name=subtextai-ec2-profile` (see [Backend Deployment](#backend-deployment) below), so every container running on it inherits these permissions automatically — no key file, no `aws configure` on the box.

---

## Why pgvector Rather Than a Managed Vector Search Service

Managed vector search services (e.g., Amazon OpenSearch with the k-NN plugin) offer hybrid retrieval and reranking as built-in features. pgvector does not — it provides vector similarity only, and the hybrid layer has to be built explicitly (see [AI Pipeline](ai-pipeline.md) for the SQL and fusion logic).

pgvector was chosen anyway, because it removes an entire service from the topology: traces, vectors, and audit records live in one PostgreSQL container with one consistency model and one backup story. Running that container on the same EC2 instance as the backend removes a second billed resource on top of removing a second service — the cost is roughly 200 lines of retrieval code plus owning your own backup schedule, which is a fair trade at this scale.

A managed instance (RDS) or a managed vector search service (OpenSearch) becomes the better option once corpus size, query volume, or uptime requirements outgrow what a single EC2 instance handles comfortably — trading self-managed backups and single points of failure for a managed feature, at the cost of a recurring bill.

---

## Database Setup on Docker

Unlike RDS, there's no parameter group to configure — the `vector` extension is enabled directly against a Postgres image that already ships with it (e.g., `pgvector/pgvector:pg16`):

```yaml
# docker-compose.yml (excerpt)
services:
  postgres:
    image: pgvector/pgvector:pg16
    restart: unless-stopped
    environment:
      POSTGRES_DB: subtextai
      POSTGRES_USER: subtextai_api
      POSTGRES_PASSWORD_FILE: /run/secrets/postgres_password
    volumes:
      - postgres_data:/var/lib/postgresql/data
    networks:
      - internal
```

Then, inside the database:

```sql
CREATE EXTENSION IF NOT EXISTS vector;
```

The same HNSW and GIN indexes described in [AI Pipeline](ai-pipeline.md) apply unchanged.

**Backup responsibility.** RDS gives automated snapshots for free; a Dockerized Postgres does not. A scheduled `pg_dump` to the S3 bucket already provisioned for the corpus (via a small cron job on the instance, authenticated through the same IAM instance profile) covers this at negligible cost — see [Deployment](deployment.md) for the script.

---

## Backend Deployment

### Amazon EC2 (Free Tier — `t3.micro`, Docker Compose)

**1. Build and push the image to Amazon ECR:**

```bash
docker build -f docker/Dockerfile.backend -t subtextai-api .

aws ecr get-login-password --region <region> | docker login --username AWS --password-stdin <account-id>.dkr.ecr.<region>.amazonaws.com
docker tag subtextai-api <account-id>.dkr.ecr.<region>.amazonaws.com/subtextai-api
docker push <account-id>.dkr.ecr.<region>.amazonaws.com/subtextai-api
```

**2. Launch the EC2 instance:**

```bash
aws ec2 run-instances \
  --image-id ami-0abcdef1234567890 \
  --instance-type t3.micro \
  --key-name subtextai-key \
  --iam-instance-profile Name=subtextai-ec2-profile \
  --security-group-ids <sg-id> \
  --subnet-id <subnet-id> \
  --block-device-mappings '[{"DeviceName":"/dev/xvda","Ebs":{"VolumeSize":20,"VolumeType":"gp3"}}]' \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=subtextai-api}]'
```

**3. Install Docker and pull the image:**

```bash
ssh -i subtextai-key.pem ec2-user@<instance-public-ip>

sudo yum update -y
sudo yum install -y docker
sudo systemctl enable --now docker
sudo usermod -aG docker ec2-user

aws ecr get-login-password --region <region> | docker login --username AWS --password-stdin <account-id>.dkr.ecr.<region>.amazonaws.com
docker pull <account-id>.dkr.ecr.<region>.amazonaws.com/subtextai-api
```

**4. Run the stack with Docker Compose** (backend, Postgres, Redis, Nginx, all on the same instance, on an internal Docker network so only Nginx is publicly reachable):

```yaml
# docker-compose.yml
services:
  api:
    image: <account-id>.dkr.ecr.<region>.amazonaws.com/subtextai-api
    restart: unless-stopped
    env_file: .env
    networks: [internal]
    expose: ["8000"]

  postgres:
    image: pgvector/pgvector:pg16
    restart: unless-stopped
    volumes: [postgres_data:/var/lib/postgresql/data]
    networks: [internal]

  redis:
    image: redis:7-alpine
    restart: unless-stopped
    volumes: [redis_data:/data]
    networks: [internal]

  nginx:
    image: nginx:alpine
    restart: unless-stopped
    ports: ["80:80", "443:443"]
    volumes:
      - ./nginx.conf:/etc/nginx/conf.d/default.conf:ro
      - ./certs:/etc/nginx/certs:ro
    depends_on: [api]
    networks: [internal]

networks:
  internal:

volumes:
  postgres_data:
  redis_data:
```

```bash
docker compose up -d
```

**5. Configure Nginx as the reverse proxy** (`nginx.conf`):

```nginx
server {
    listen 443 ssl;
    server_name api.subtextai.example;

    ssl_certificate     /etc/nginx/certs/fullchain.pem;
    ssl_certificate_key /etc/nginx/certs/privkey.pem;

    location / {
        proxy_pass http://api:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}

server {
    listen 80;
    server_name api.subtextai.example;
    return 301 https://$host$request_uri;
}
```

**Single-instance caveat.** There is no auto-scaling and no automatic failover — if the instance stops, the API stops. That's the honest cost of Free Tier hosting versus App Runner or ECS Fargate, both of which handle instance failure transparently. A CloudWatch alarm on instance status (see [Observability](#observability)) is the practical mitigation at this scale: it pages before a user notices, even though it doesn't self-heal.

### Frontend → AWS Amplify Hosting

```bash
cd frontend
npm run build

aws amplify create-app --name subtextai-frontend --repository <github-repo-url>
aws amplify create-branch --app-id <app-id> --branch-name main
aws amplify start-deployment --app-id <app-id> --branch-name main
```

Update `VITE_API_BASE_URL` in the Amplify app's environment variables to point to the EC2 instance's domain (through Nginx, over HTTPS).

### CI/CD with GitHub Actions

The `.github/workflows/` directory adds pipelines for:
- Backend: lint (Ruff), test (Pytest), build image, push to ECR, SSH into the EC2 instance and re-pull + `docker compose up -d`
- Frontend: lint, test, build, deploy to Amplify
- Scheduled: evaluation harness run, publishing metrics to CloudWatch

Workflows authenticate to AWS with **OIDC federated credentials** via `aws-actions/configure-aws-credentials`, so no long-lived IAM access key is stored in GitHub. The SSH step uses a deploy key stored in GitHub Secrets, scoped only to the deployment user on the instance.

---

## Capacity Planning: Two Ceilings Now, Not One

There are two constraints stacked on top of each other in this topology, where a managed deployment would only have the first.

**1. OpenRouter's rate limit and per-token cost** — unchanged from a managed deployment:

| Call | Approximate tokens |
|-|-|
| Crisis classifier | ~200 in / ~20 out |
| Pragmatic analysis | ~2,000 in (prompt + retrieved fragments) / ~400 out |
| **Per request** | **~2,600 tokens** |

**2. `t3.micro` compute and memory** — new in this topology. `t3.micro` has 2 vCPUs (burstable, not sustained) and 1GB RAM, shared across the FastAPI process, Postgres, Redis, and Nginx simultaneously. This is comfortably enough for low concurrent traffic (a portfolio demo, a handful of simultaneous users) but is the binding constraint before OpenRouter's own limits are ever reached at meaningful scale. CPU credit exhaustion under sustained load — not token quota — is the first thing to monitor.

Mitigations, in order of impact:
1. Semantic caching (see [Roadmap](roadmap.md)) — avoids the analysis call entirely on equivalent messages, reducing both OpenRouter cost and instance CPU load
2. Trimming retrieved fragment count, which dominates input tokens and reranker CPU time
3. Moving Redis or Postgres to their managed equivalents first (ElastiCache, RDS) if the instance becomes memory-bound before it becomes traffic-bound — cheaper than upsizing the instance type in some cases
4. Upsizing to `t3.small` (still cheap, no longer free) once sustained concurrent traffic exceeds what CPU credits absorb

Monitor CPU credit balance (`CPUCreditBalance` in CloudWatch) alongside OpenRouter's usage dashboard and the `429` rate.

---

## Observability

| Layer | Technology |
|-|-|
| **Distributed tracing** | OpenTelemetry, exported to AWS X-Ray |
| **Metrics & alerts** | Amazon CloudWatch, via the CloudWatch agent installed on the EC2 instance |
| **Logging** | Structured logging, shipped to CloudWatch Logs by the CloudWatch agent |
| **Health checks** | `/health` endpoint monitoring the local Postgres and Redis containers plus the OpenRouter API |

Unlike App Runner or ECS, EC2 does not ship logs and metrics to CloudWatch automatically — the **CloudWatch agent** must be installed and configured on the instance (covered by the `CloudWatchAgentServerPolicy` already attached to the instance profile). Once running, it aggregates metrics from both production interactions and the evaluation job, enabling the same behavior monitoring and prompt version comparison a managed deployment would give.

**Capacity signal worth watching:** `CPUCreditBalance` and `MemoryUtilization` on the instance, alongside OpenRouter's own rate limit — see [Capacity Planning](#capacity-planning-two-ceilings-now-not-one).

---

## Data Residency and Compliance Notes

The [GDPR-aware storage design](design-principles.md#traceability-under-data-protection) (split storage, text-level erasure, retention TTL) does not change based on hosting cost — it is implemented in the backend, independent of where Postgres happens to run. What does change:

- **Region.** The EC2 instance, S3 bucket, and Secrets Manager secret should all sit in a single, explicitly configured region matching where users are located.
- **OpenRouter is a third party.** OpenRouter is an external router that forwards requests to whichever underlying provider serves the selected model. Data residency and no-training guarantees are only as strong as OpenRouter's terms **and** the terms of the specific model it routes to — this needs explicit verification per model, not an assumption inherited from AWS's own compliance posture.
- **Self-managed database means self-managed backup encryption.** RDS encrypts automated backups by default; a `pg_dump` written to S3 must be encrypted explicitly (SSE-S3 or SSE-KMS on the bucket) since that responsibility no longer belongs to a managed service.

---

## Environment Variables (AWS)

| Variable | Description |
|-|-|
| `OPENROUTER_API_KEY` | Resolved from Secrets Manager at startup, never set directly in the environment |
| `OPENROUTER_MODEL` | Generation model route (`openai/gpt-4.1`) |
| `OPENROUTER_EMBEDDING_MODEL` | Embedding model route (`text-embedding-3-large`) |
| `RERANK_MODEL` / `RERANK_ENDPOINT` | Reranker identity and endpoint, if also hosted on AWS |
| `POSTGRES_HOST` | `postgres` (the Docker Compose service name — same-host container-to-container networking, not a public endpoint) |
| `POSTGRES_DB` / `POSTGRES_USER` | Database connection details for the local Postgres container |
| `REDIS_URL` | `redis://redis:6379/0` (the Docker Compose service name) |
| `AWS_REGION` | Region for all AWS SDK calls (Secrets Manager, S3, CloudWatch — resolved via the instance's IAM role) |
| `COGNITO_USER_POOL_ID` / `COGNITO_CLIENT_ID` | Cognito user pool and app client for OAuth2 token validation |

Policy thresholds (`RERANK_SCORE_THRESHOLD_HIGH`, `RERANK_SCORE_THRESHOLD_LOW`, `RATE_LIMIT_PER_MINUTE`, `RAW_TEXT_RETENTION_DAYS`, `MIN_WORD_COUNT`) are unchanged — those are pipeline behavior, not infrastructure.
