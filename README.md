# SubtextAI

> Understand what people really mean.

AI-powered communication intelligence platform that helps people understand intent, emotions, and hidden meaning behind ambiguous conversations.

`Python` · `FastAPI` · `React 19` · `Ollama` · `PostgreSQL + pgvector` · `Docker`

---

## Project Status

🚧 **In active development** — architecture and design complete; backend implementation (policy engine, retrieval, reranking, traceability) in progress on the local Ollama stack.

---

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

```
  Frontend (React 19)        Backend (FastAPI)            AI & Data
 ┌──────────────────┐    ┌──────────────────────┐    ┌──────────────────┐
 │ Local dev server │    │  Local (Docker/       │    │ Ollama           │
 │  (Vite)          │───▶│  Uvicorn)             │───▶│  llama3.2:3b     │
 │                  │REST│  Governance Pipeline  │    │  nomic-embed-text│
 │ TypeScript       │API │  ┌──────────────────┐ │    └──────────────────┘
 │ Tailwind CSS     │    │  │ 1. Policy        │ │    ┌──────────────────┐
 │ shadcn/ui        │    │  │ 2. Crisis ‖ RAG  │ │    │ PostgreSQL       │
 │ Recharts         │    │  │ 3. Rerank + gate │ │───▶│  + pgvector      │
 │ Framer Motion    │    │  │ 4. Analysis      │ │    │  HNSW + GIN      │
 └──────────────────┘    │  │ 5. Trace         │ │    └──────────────────┘
                         │  └──────────────────┘ │    ┌──────────────────┐
                         │  LangGraph            │    │ Redis            │
                         │  No API keys, no cloud│    │  (cache/sessions)│
                         └───────────────────────┘    └──────────────────┘
                                   │
                                   ▼
                          Local structured logs + OpenTelemetry
                          (console exporter)
```

Everything above runs on your machine — no cloud account, no API keys, no data leaving the device. Steps 2a (crisis detection) and 2b (retrieval) run concurrently — crisis checking stays strictly blocking, but leaves the critical path of a successful request. The confidence gate at step 3 reads a calibrated cross-encoder score, so generation is blocked whenever no solid evidence was found.

> Full architecture documentation in [docs/architecture.md](docs/architecture.md)

---

## Tech Stack

| Layer | Technology |
|-|-|
| **Frontend** | React 19, TypeScript, Vite, Tailwind CSS, shadcn/ui, Recharts, Framer Motion |
| **Backend** | Python 3.13, FastAPI, Pydantic v2, SQLAlchemy 2.0, Alembic |
| **AI** | Ollama (`llama3.2:3b`, `nomic-embed-text`), LangGraph, Structured Outputs |
| **RAG** | PostgreSQL + pgvector (HNSW), lexical search, RRF fusion, cross-encoder reranking |
| **Data** | PostgreSQL + pgvector (Docker), Redis |
| **Security** | JWT (self-issued), rate limiting, prompt injection detection |
| **Observability** | OpenTelemetry, structured logging |
| **DevOps** | Docker, Docker Compose, GitHub Actions (lint + test), Ruff, Pytest |

> Detail and rationale in [docs/tech-stack.md](docs/tech-stack.md)

---

## Quick Start

### Prerequisites

* Python 3.13+
* Node.js 18+ and npm
* Docker and Docker Compose
* [Ollama](https://ollama.com) installed, with `llama3.2:3b` and `nomic-embed-text` pulled

### With Docker (recommended)

```bash
git clone https://github.com/<your-user>/subtextai.git
cd subtextai

ollama pull llama3.2:3b
ollama pull nomic-embed-text

cp .env.example .env        # Local endpoints and configuration only — no keys

docker compose up --build
# Backend  → http://localhost:8000
# Frontend → http://localhost:5173
```

> The application stores **no API keys** and makes **no calls to any external service**. Everything — model inference, embeddings, database, cache — runs on your machine via Ollama and Docker.

### Manual setup

```bash
# Backend
cd backend
python -m venv .venv
source .venv/bin/activate        # Windows: .venv\Scripts\activate
pip install -r requirements.txt
alembic upgrade head
uvicorn main:app --reload --port 8000

# Frontend (separate terminal)
cd frontend
npm install
cp .env.example .env.local
# Edit .env.local → VITE_API_BASE_URL=http://localhost:8000/api/v1
npm run dev                      # → http://localhost:5173
```

> Full installation and deployment guide in [docs/deployment.md](docs/deployment.md)

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
- [Cloud Deployment (Azure, optional/paid)](docs/cloud-deployment-azure.md)

---

## Roadmap

### Vision

Become the communication intelligence platform for personal and enterprise conversations.

| Phase | Description |
|-|-|
| **Current** | Architecture and governance pipeline fully designed; backend implementation (policy engine, retrieval, reranking, traceability) in progress on the local Ollama stack |
| **Next** | Specialized crisis classifier, multilingual support, semantic cache, audit panel |
| **Future** | Real-time streaming analysis, predictive trajectory engine |

> Detail in [docs/roadmap.md](docs/roadmap.md)

---

## Author

**Heily Madelay Tandazo**

## License

Distributed under the MIT License — see [LICENSE](LICENSE) for details.
