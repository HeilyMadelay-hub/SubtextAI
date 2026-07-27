# API Reference

**Base URL (production):** `https://<app-runner-service-url>/api/v1`
**Base URL (dev, pointed at AWS resources):** `http://localhost:8000/api/v1`

Built with **FastAPI** (Python 3.13). Interactive API docs available at `/docs` (Swagger UI) and `/redoc` (ReDoc).

---

## Endpoints

| Method | Endpoint | Description |
|-|-|-|
| `POST` | `/analyze` | Interprets the pragmatic meaning of an ambiguous message |
| `POST` | `/evaluate` | Evaluates a user-proposed response to an analyzed message |
| `POST` | `/draft` | Generates a suggested reply for an analyzed message |
| `GET` | `/replay/{trace_id}` | Returns the analyzed and annotated conversation for Replay Mode |
| `GET` | `/audit/{trace_id}` | Returns complete, versioned traceability of a response |
| `DELETE` | `/messages/{trace_id}/content` | Erases raw message text, preserving the audit trail |
| `GET` | `/metrics` | Aggregated system quality metrics (last 7 days) |
| `GET` | `/prompts` | Prompt version history with comparative metrics |
| `GET` | `/health` | System status and connected AWS services |

---

## POST /analyze

Receives a message and context, passes it through the complete governance pipeline (orchestrated by LangGraph), and returns the pragmatic analysis grounded in documentary sources.

### Request

```json
{
  "message": "Solo quiero fluir y ver qué pasa",
  "context": "relationship | work | social | negotiation"
}
```

### Response 200

```json
{
  "meaning": "Low emotional involvement, explicit avoidance of commitment",
  "signals": ["avoidance", "intentional ambiguity", "passivity"],
  "alert_level": "MEDIUM",
  "recommendation": "Possible avoidance of commitment. Consider clarifying expectations.",
  "source": {
    "document": "<source document title>",
    "fragment": "<relevant section or chapter>"
  },
  "confidence": {
    "level": "HIGH",
    "reason": "top_rerank_score: 0.84 — strong documentary context"
  },
  "grounded": true,
  "applied_policy": "none",
  "detected_language": "es",
  "trace_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "metadata": {
    "latency_ms": 2310,
    "model": "openai/gpt-4.1",
    "prompt_version": "v1.2",
    "embedding_model": "text-embedding-3-large",
    "rerank_model": "cross-encoder/ms-marco-MiniLM-L-6-v2",
    "from_cache": false,
    "sentiment": { "label": "neutral", "score": 0.62 }
  }
}
```

The confidence level is derived from the **top cross-encoder reranker score**, not from raw similarity or the RRF fusion score. See [AI Pipeline](ai-pipeline.md) for the rationale.

Latency is dominated by the main model call and depends on OpenRouter's routing overhead plus the underlying model provider — see [AI Pipeline](ai-pipeline.md) for the benchmarking note.

### Error Codes

| Code | Policy | Description |
|-|-|-|
| `422` | `minimum_message` | Message with fewer than 5 words |
| `422` | `mandatory_grounding` | Top reranker score < 0.45; insufficient documentary basis |
| `422` | `crisis_detected` | Emotional crisis signals; redirects to a professional |
| `422` | `unsupported_language` | Language other than the configured one |
| `400` | `prompt_injection` | Malicious input detected and blocked |
| `429` | `user_rate_limit` | More than 10 requests per minute per user/IP |

> A `429` may also originate from OpenRouter itself if its per-key rate limit is hit under concurrent load. The response distinguishes the two cases via `applied_policy`.

### Crisis Response

`crisis_detected` is the one policy where the response body itself matters as much as the status code — it is the moment the system hands a real person off to real help. The body is not a generic error; it carries an explicit support resource and disclaimer:

```json
{
  "applied_policy": "crisis_detected",
  "grounded": false,
  "message": "Este sistema no está diseñado para gestionar situaciones de crisis y no sustituye la ayuda profesional.",
  "support_resource": {
    "name": "Teléfono de la Esperanza",
    "phone": "717 003 717",
    "available": "24/7"
  },
  "disclaimer": "SubtextAI is a communication analysis tool, not a crisis service or a licensed therapist.",
  "trace_id": "c3d4e5f6-a7b8-9012-cdef-123456789012"
}
```

The support resource is configurable per deployment region — a system serving users outside Spain must be configured with a locally valid crisis line before launch, not the placeholder above. This is a deployment-blocking requirement, not a cosmetic default.

---

## POST /evaluate

Evaluates a user-proposed response to an already-analyzed message: success probability, strengths, areas for improvement, and an alternative suggestion — all grounded in sources.

---

## POST /draft

Generates a ready-to-send reply for an already-analyzed message. Unlike `/evaluate`, which scores a response the user already wrote, `/draft` writes the response itself.

It runs through the same governance pipeline as `/analyze` — mandatory grounding, policy validation, traceability — with the interpretation from the referenced `trace_id` as additional context. No new model call type is introduced; this is the same LLM with a different prompt.

### Request

```json
{
  "trace_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "tone": "conciliatory | assertive | neutral",
  "goal": "clarify | close | reconnect"
}
```

### Response 200

```json
{
  "suggested_message": "Entiendo que prefieres no forzar las cosas, pero para mí es importante saber si esto va en serio. ¿Podemos hablarlo con calma?",
  "explanation": "This message avoids sounding accusatory, validates their position, and asks for clarity without pressure.",
  "source_trace_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "trace_id": "b2c3d4e5-f6a7-8901-bcde-f12345678901",
  "grounded": true,
  "metadata": {
    "latency_ms": 1850,
    "model": "llama3.2:3b",
    "prompt_version": "v1.0",
    "from_cache": false
  }
}
```

`source_trace_id` links back to the original analysis; `trace_id` identifies this draft's own audit record, since generating a reply is itself a governed, traceable interaction.

### Error Codes

Same policy set as `/analyze` (`crisis_detected`, `prompt_injection`, `user_rate_limit`), plus:

| Code | Policy | Description |
|-|-|-|
| `404` | `trace_not_found` | The referenced `trace_id` does not exist or has been redacted |
| `422` | `harmful_draft_detected` | The generated reply itself was flagged as manipulative or coercive and withheld |

`harmful_draft_detected` validates the model's **output**, not the request — every other policy in this API checks what the user sent in; this one checks what the system is about to say on the user's behalf before it goes out. See [AI Pipeline](ai-pipeline.md#step-5--output-validation) for the rationale.

---

## GET /replay/{trace_id}

Returns the annotated conversation for Replay Mode:
- Per-message annotations (emotional intensity, intent change, subtext, confidence)
- Critical moments (first ambiguity, first escalation, first contradiction, conflict trigger)
- Tension meter evolution
- Temporal tension heatmap data
- Explanations for each inflection point

All information comes from already-registered traces.

---

## GET /audit/{trace_id}

Complete record of an interaction:
- Retrieved fragments with vector score, lexical score, RRF rank, and reranker score
- Embedding and reranker model identity and version
- Exact prompt and version (`prompt_version`)
- Model and version
- Evaluated policies
- Per-stage and total latency

Model identities are persisted so that threshold drift is detectable: if the reranker or embedding model changes without recalibration, the audit record shows it.

If the raw message text has been erased or has passed its retention TTL, the record returns `content_redacted: true` and the text field is null. Every other field remains intact — the decision chain is still fully reconstructible.

---

## DELETE /messages/{trace_id}/content

Erases the raw message text for a given trace while preserving the annotation layer.

Implements the right to erasure (GDPR Art. 17) without breaking the audit trail: scores, policy decisions, model versions, and derived annotations survive. Replay Mode continues to work with the content redacted.

### Response 200

```json
{
  "trace_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "content_redacted": true,
  "annotations_preserved": true,
  "redacted_at": "2026-07-25T10:14:22Z"
}
```

Raw text is also purged automatically once it exceeds `RAW_TEXT_RETENTION_DAYS` (default 90). See [Design Principles](design-principles.md) for the full data-protection model.

---

## GET /metrics

Aggregated metrics for the last 7 days.

---

## GET /prompts

Prompt version history with per-version metrics. Enables before/after comparison of each prompt change.

---

## GET /health

System status and each connected service (OpenRouter, RDS PostgreSQL + pgvector, ElastiCache for Redis).

---

## Authentication

API requests require a valid **OAuth2 access token** issued by **Amazon Cognito**. The backend validates the token's signature and claims on every request rather than issuing its own — Cognito owns the user pool, sign-up/sign-in flows, and MFA.

Calls between the backend and RDS, ElastiCache, S3, and CloudWatch authenticate via the backend's **IAM role** — short-lived credentials, nothing stored in configuration. The one exception is OpenRouter, which sits outside AWS's IAM boundary: its API key is resolved from **Secrets Manager** at startup rather than passed as a request-time credential.

---

## WebSocket

Optional WebSocket connection at `/ws/session/{session_id}` for real-time telemetry updates during live analysis sessions.
