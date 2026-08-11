# SubtextAI

> Understand what people really mean.

AI-powered communication intelligence platform that helps people understand intent, emotions, and hidden meaning behind ambiguous conversations.

`Python` · `FastAPI` · `React 19` · `AWS` · `OpenRouter` · `PostgreSQL + pgvector` · `Docker`

## Screenshots

<!-- Replace with actual screenshots -->

| Analysis | Replay Mode | Live Telemetry | Audit |
|-|-|-|-|
| ![Analysis](docs/screenshots/analysis.png) | ![Replay](docs/screenshots/replay.png) | ![Telemetry](docs/screenshots/telemetry.png) | ![Audit](docs/screenshots/audit.png) |

---

## Demo

<!-- Replace with ~20 second GIF showing the full flow -->

![SubtextAI Demo](docs/screenshots/demo.gif)

---

## Overview

SubtextAI is a communication intelligence engine that analyzes ambiguous conversations across real-world contexts — **relationships, work, social settings, and negotiation** — and turns them into structured, evidence-based insights.

It detects intent shifts, emotional intensity, and the gap between what's said and what's meant. Every interpretation is grounded in documentary sources, scored with an objective confidence level, and fully auditable by `trace_id`.

---

## The Problem

Most misunderstandings aren't about what's said — they're about what's meant.

Unlike a conventional assistant, **SubtextAI is not a black box**: every interpretation is grounded in real documentary sources, calculated with an objective confidence level, and fully auditable. It's not an academic chatbot — it's a conversational behavior interpretation tool.

---

## Why SubtextAI?

Unlike traditional AI assistants, SubtextAI doesn't just generate answers.

It explains *why* a message may be interpreted in a certain way, grounds every conclusion on evidence, and helps users make better communication decisions.

---

## Philosophy

AI should improve human communication, not replace it.

Every insight should be explainable.

Every response should be grounded.

Every decision should be auditable.

No interpretation should be presented as absolute truth.

This system does not replace professional advice, therapy, or legal counsel.

---

## Key Features

**Conversation Analysis** — Paste any ambiguous message with its context and get a complete pragmatic analysis: what was said vs. what was meant.

**Intent Detection** — Identifies intent transitions throughout the conversation (e.g., `neutral → defensive → aggressive`).

**Emotion Detection** — Tracks emotional intensity per message (low / medium / high) and detects escalation patterns.

**Hidden Meaning** — Surfaces the gap between explicit and implicit meaning — the subtext that drives miscommunication.

**Response Evaluation** — Scores a response the user already wrote: success probability, strengths, areas for improvement, and an evidence-based alternative.

**Reply Drafting** — When the user doesn't know what to say, the system writes the reply itself — grounded in the same sources as the original analysis, with a chosen tone and goal.

**Replay Mode** — Transforms any analyzed conversation into an interactive timeline of emotion, intent, and tension — like a replay of human decisions. Includes autoplay demo with pre-recorded conversations.

**Live Session Telemetry** — Real-time panel tracking emotional velocity, conflict acceleration, and accumulated tension as the conversation unfolds, message by message.

---

## How It Works

```
Paste a conversation
        ↓
AI understands the context
        ↓
Relevant evidence is retrieved
        ↓
Conversation is analysed
        ↓
Insights are generated
        ↓
Suggested response
```

> Full technical pipeline in [docs/ai-pipeline.md](docs/ai-pipeline.md)

---

## Why SubtextAI is Different

| Principle | What it means |
|-|-|
| **Explainable AI** | Every interpretation is grounded in documentary sources. No evidence, no response. |
| **Enterprise Governance** | Security rules live in the pipeline, not in the prompt — the model never executes if a critical policy is violated. |
| **Full Traceability** | Every response is reconstructible via `trace_id`: documents, scores, prompt version, model, and policies evaluated. |
| **Privacy by Design** | Raw text and audit annotations are stored separately, so erasure requests and retention limits never break the audit trail. |

> Learn more in [docs/design-principles.md](docs/design-principles.md)

---

## Architecture

<p align="center"> <img src="docs/screenshots/arquitectura.png" alt="Narek Architecture" width="900"> </p>

Everything above runs on AWS, sized to fit inside the Free Tier — a single EC2 instance runs the entire backend stack in Docker (FastAPI, PostgreSQL + pgvector, Redis, Nginx), with only the frontend, auth, storage, secrets, and observability handled by separate managed services. Steps 2a (crisis detection) and 2b (retrieval) run concurrently — crisis checking stays strictly blocking, but leaves the critical path of a successful request. The confidence gate at step 3 reads a calibrated cross-encoder score, so generation is blocked whenever no solid evidence was found.

> Full architecture documentation in [docs/architecture.md](docs/architecture.md)

---

## Tech Stack

| Layer | Technology |
|-|-|
| **Frontend** | React 19, TypeScript, Vite, Tailwind CSS, shadcn/ui, Recharts, Framer Motion |
| **Backend** | Python 3.13, FastAPI, Pydantic v2, SQLAlchemy 2.0, Alembic |
| **AI** | OpenRouter (`gpt-4.1`, `text-embedding-3-large`), LangGraph, Structured Outputs |
| **RAG** | PostgreSQL + pgvector (HNSW), lexical search, RRF fusion, cross-encoder reranking |
| **Data** | PostgreSQL + pgvector and Redis, both in Docker containers on the same Amazon EC2 instance |
| **Security** | Amazon Cognito (OAuth2 + IAM), rate limiting, prompt injection detection |
| **Observability** | Amazon CloudWatch, AWS X-Ray, OpenTelemetry |
| **DevOps** | Docker, Amazon EC2 (Free Tier `t3.micro`), Nginx, Amazon ECR, AWS Amplify Hosting, GitHub Actions (OIDC to AWS), Ruff, Pytest |

> Detail and rationale in [docs/tech-stack.md](docs/tech-stack.md)

---

## Deployment

SubtextAI deploys to AWS, sized to run within the **Free Tier**: a single **Amazon EC2** `t3.micro` instance runs the whole backend stack in Docker — **FastAPI**, **PostgreSQL + pgvector**, **Redis**, and **Nginx** as the reverse proxy — fronted by **Amplify Hosting** (frontend), **S3** for the document corpus, and **OpenRouter** for generation (`gpt-4.1`) and embeddings (`text-embedding-3-large`).

### Prerequisites

* Python 3.13+
* Node.js 18+ and npm
* Docker (for building the backend image)
* An AWS account with the AWS CLI configured, and an EC2 key pair
* An [OpenRouter](https://openrouter.ai) API key

### Build and push the backend image to ECR

```bash
git clone https://github.com/<your-user>/subtextai.git
cd subtextai

docker build -f docker/Dockerfile.backend -t subtextai-api .
aws ecr get-login-password --region <region> | docker login --username AWS --password-stdin <account-id>.dkr.ecr.<region>.amazonaws.com
docker tag subtextai-api <account-id>.dkr.ecr.<region>.amazonaws.com/subtextai-api
docker push <account-id>.dkr.ecr.<region>.amazonaws.com/subtextai-api
```

### Store the OpenRouter key

```bash
aws secretsmanager create-secret \
  --name subtextai/openrouter-api-key \
  --secret-string '{"api_key":"<your-openrouter-key>"}'
```

### Launch EC2 and run the stack with Docker Compose

```bash
aws ec2 run-instances \
  --image-id ami-0abcdef1234567890 \
  --instance-type t3.micro \
  --key-name subtextai-key \
  --iam-instance-profile Name=subtextai-ec2-profile \
  --security-group-ids <sg-id>

ssh -i subtextai-key.pem ec2-user@<instance-public-ip>
sudo yum install -y docker && sudo systemctl enable --now docker
aws ecr get-login-password --region <region> | docker login --username AWS --password-stdin <account-id>.dkr.ecr.<region>.amazonaws.com
docker pull <account-id>.dkr.ecr.<region>.amazonaws.com/subtextai-api

# docker-compose.yml runs api + postgres (pgvector) + redis + nginx together
docker compose up -d
cd backend && alembic upgrade head   # run against the local Postgres container
```

### Deploy the frontend

```bash
cd frontend
npm run build
aws amplify start-deployment --app-id <app-id> --branch-name main
```

> Full infrastructure setup (IAM instance profile, Nginx config, backup strategy) and env vars in [docs/cloud-deployment-aws.md](docs/cloud-deployment-aws.md) and [docs/deployment.md](docs/deployment.md)

---

## Documentation

- [Architecture](docs/architecture.md)
- [AI Pipeline](docs/ai-pipeline.md)
- [Design Principles](docs/design-principles.md)
- [Technology Stack](docs/tech-stack.md)
- [Testing Strategy](docs/testing-strategy.md)
- [Deployment Guide](docs/deployment.md)
- [API Reference](docs/api-reference.md)
- [Screenshots](docs/screenshots.md)
- [Roadmap](docs/roadmap.md)
- [Cloud Deployment (AWS)](docs/cloud-deployment-aws.md)

---

## Roadmap

### Vision

Become the communication intelligence platform for personal and enterprise conversations.

| Phase | Description |
|-|-|
| **Current** | Architecture and governance pipeline fully designed; backend implementation (policy engine, retrieval, reranking, traceability) in progress, targeting production on AWS |
| **Next** | Specialized crisis classifier, multilingual support, semantic cache, audit panel |
| **Future** | Real-time streaming analysis, predictive trajectory engine |

> Detail in [docs/roadmap.md](docs/roadmap.md)

---

## Author

**Heily Madelay Tandazo**

## License

Distributed under the MIT License — see [LICENSE](LICENSE) for details.
