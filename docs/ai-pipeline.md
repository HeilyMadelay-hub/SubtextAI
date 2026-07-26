# AI Pipeline

## Cascade Short-Circuit Pipeline

If any policy triggers at any step, the flow stops and the main model is never invoked. The order is strict:

```
1. Policy Validation (no LLM)
   → min length, language, rate limit, prompt injection
              ↓
2. Parallel execution
   ├── Crisis Classifier (LLM call #1)
   │   → Ollama (llama3.2:3b), specialized prompt (~150-200 tokens)
   └── Retrieval (embedding + hybrid search)
       → Ollama (nomic-embed-text) + PostgreSQL/pgvector
              ↓
3. Reranking + Confidence Gate
   → Cross-encoder reranking, calibrated score threshold
              ↓
4. Main LLM — Pragmatic Analysis (LLM call #2)
   → Ollama (llama3.2:3b) with mandatory grounding
              ↓
5. Traceability
   → Full record in PostgreSQL with unique trace_id
```

The pipeline is orchestrated with **LangGraph**. Steps 2a and 2b run as parallel nodes: if the crisis classifier fires, the retrieval branch is cancelled and the request short-circuits with a `422`. This keeps crisis detection strictly blocking while removing it from the critical path of the happy case, which represents the vast majority of requests.

All inputs and outputs are validated with **Pydantic v2** and **Structured Outputs** for typed JSON responses.

---

## Step 1 — Policy Validation (no LLM)

Hard rules coded in the pipeline (not in the prompt). The model is never invoked if any rule is violated, so security behavior does not depend on the model or its configuration.

| Policy | Trigger condition | Action | HTTP |
|-|-|-|-|
| `minimum_message` | Fewer than 5 words | Reject and request more context | 422 |
| `mandatory_grounding` | Reranker top score below threshold | Block generation without documentary basis | 422 |
| `crisis_detected` | Emotional crisis signals | Block and redirect to a professional | 422 |
| `prompt_injection` | Malicious input detected | Block and log to audit | 400 |
| `unsupported_language` | Language other than configured | Reject informing detected language | 422 |
| `user_rate_limit` | > 10 req/min per user/IP | Temporarily block with logging | 429 |
| `response_without_source` | Response without valid documentary evidence | Block before delivery | 422 |
| `insufficient_confidence` | Reranker score below threshold | Reject generation | 422 |
| `policy_conflict` | Internal inconsistency between policies | Block flow | 500 |

---

## Step 2a — Emotional Crisis Classifier

An independent model call (LLM call #1) with a specialized, minimalist prompt (~150-200 tokens) evaluates whether the message contains signals of severe crisis.

A prompt-based classifier was chosen — instead of a fine-tuned model or keyword list — prioritizing architectural coherence, auditability, and development speed in the context of an MVP, with its future evolution documented.

This step runs **in parallel** with retrieval but remains strictly blocking: if `crisis_detected` fires, the retrieval result is discarded and the main model is never invoked. Parallelism removes latency from the happy path without weakening the guarantee.

Each activation is recorded with its `trace_id` and severity level.

---

## Step 2b — Retrieval: Hybrid Search

### Indexing Pipeline

Documents are split into chunks, vectorized locally with **Ollama** (`nomic-embed-text`), and stored in **PostgreSQL + pgvector**. Each chunk row holds both the embedding vector and a `tsvector` column for lexical search.

```sql
CREATE TABLE document_chunks (
    id            BIGSERIAL PRIMARY KEY,
    document_id   BIGINT NOT NULL REFERENCES documents(id),
    content       TEXT NOT NULL,
    embedding     vector(768) NOT NULL,
    content_tsv   tsvector GENERATED ALWAYS AS (
                      to_tsvector('spanish', content)
                  ) STORED,
    metadata      JSONB
);

CREATE INDEX ON document_chunks
    USING hnsw (embedding vector_cosine_ops)
    WITH (m = 16, ef_construction = 64);

CREATE INDEX ON document_chunks USING GIN (content_tsv);
```

### Hybrid Search Implementation

pgvector provides vector similarity only. Hybrid retrieval is implemented explicitly in three stages:

**1. Vector branch** — cosine distance over the HNSW index:

```sql
SELECT id, 1 - (embedding <=> :query_vector) AS score
FROM document_chunks
ORDER BY embedding <=> :query_vector
LIMIT 50;
```

**2. Lexical branch** — BM25-style ranking over the GIN index:

```sql
SELECT id, ts_rank_cd(content_tsv, plainto_tsquery('spanish', :query)) AS score
FROM document_chunks
WHERE content_tsv @@ plainto_tsquery('spanish', :query)
ORDER BY score DESC
LIMIT 50;
```

**3. Fusion** — the two ranked lists are merged with **Reciprocal Rank Fusion**:

```
RRF(d) = Σ  1 / (k + rank_i(d))        k = 60
```

RRF is rank-based, not score-based, which makes it robust to the fact that cosine similarity and `ts_rank_cd` live on incomparable scales. Its output is **not** a similarity value and must never be used as a confidence threshold — see the note below.

---

## Step 3 — Reranking and Confidence Gate

The top 20 candidates from RRF are passed to a **cross-encoder reranker**, which scores each `(query, chunk)` pair jointly and returns a calibrated relevance score in `[0, 1]`.

### Why the confidence gate uses the reranker score

Two earlier candidates were rejected as thresholding signals:

| Signal | Why it fails |
|-|-|
| **Raw cosine similarity** | Not calibrated across queries, and not comparable across embedding models. Changing the embedding model silently invalidates every threshold with no error and no alert. |
| **RRF fused score** | Rank-based, not a similarity. With `k = 60`, a first-place document scores ≈ 0.016. Any threshold expressed on a 0–1 similarity scale would reject every request. |

The cross-encoder score is calibrated, bounded, and directly comparable across queries, which makes it the only sound basis for a gate.

### Top-1, not average

The gate reads the **highest** reranker score, not the mean of retrieved fragments. Averaging punishes good retrieval: with one excellent chunk at 0.92 and four noisy ones at 0.35, the mean is 0.46 — but the system did find a strong source. Top-1 measures what actually matters: whether at least one solid piece of evidence exists.

| Level | Top reranker score | Behavior |
|-|-|-|
| **HIGH** | ≥ 0.70 | Full response with fragment and section citation. `grounded=true`. |
| **MEDIUM** | 0.45 – 0.69 | Response with partial context warning. Available source is cited. |
| **LOW** | < 0.45 | `mandatory_grounding` activates: system does not generate a response. |

> **Coupling warning.** These thresholds are calibrated against a specific reranker model. Changing the reranker requires recalibration against the evaluation set. The reranker model identity is persisted in every trace so that threshold drift is detectable in audit.

---

## Step 4 — Pragmatic Analysis (Main LLM)

The main model (Ollama, `llama3.2:3b`, LLM call #2) receives the message, context, and reranked fragments, and produces the pragmatic analysis: meaning, signals, alert level, recommendation, source, and confidence.

Grounding is mandatory: no response is delivered without documentary evidence. If the confidence gate did not pass, this step never executes.

Responses use **Structured Outputs** to guarantee typed JSON output validated by Pydantic v2.

---

## Optional Step — Reply Drafting (`/draft`)

`/analyze` interprets an incoming message; it does not write anything back. When the user wants the system to compose a reply rather than judge one, `POST /draft` reuses the same pipeline with a different terminal prompt.

```
Existing trace_id (from /analyze)
              ↓
1. Policy Validation
   → same rules as /analyze (rate limit, prompt injection)
              ↓
2. Crisis Classifier
   → re-run on the draft request itself, not skipped
              ↓
3. Load prior interpretation + retrieved sources
   → no new retrieval call; reuses the original trace's grounding
              ↓
4. Main LLM — Reply Generation
   → Ollama (llama3.2:3b), prompt conditioned on tone + goal + original grounding
              ↓
5. Output Validation
   → the generated text itself is policy-checked before being returned
              ↓
6. Traceability
   → new trace_id for the draft, linked to the source trace_id
```

Two design choices distinguish this from a plain "write me a reply" prompt:

- **Grounding is inherited, not re-fetched.** The draft is conditioned on the same retrieved fragments that grounded the original analysis, so the suggested reply is consistent with the interpretation the user already saw — it does not introduce a second, possibly contradictory, set of sources.
- **It is still a governed, audited call.** A draft gets its own `trace_id`, its own policy checks, and its own entry in `/audit`. Generating text on the user's behalf is exactly the kind of output that should not bypass governance just because it is framed as a "suggestion" rather than an "analysis."

This step does not require a new model, a new retrieval index, or any training — it is the existing pipeline with an additional terminal prompt and a linked trace.

### Step 5 — Output Validation

Every other policy in this pipeline validates the **input** — the message the system receives. A draft is different: it is text the system **produces and puts in the user's mouth**, sent to a third party under the user's name. Validating only the request that asked for it and not the reply itself would leave a real gap — a `tone: "assertive"` request could just as easily produce a firm, respectful message or a manipulative, guilt-tripping one, and only the second case is a governance failure worth catching.

The generated draft is passed through a dedicated check before being returned:

| Policy | Trigger condition | Action | HTTP |
|-|-|-|-|
| `harmful_draft_detected` | Generated text contains manipulative, guilt-tripping, or coercive language | Discard the draft and retry once with a corrective instruction; if it fails twice, return `422` | 422 |

This check runs as a lightweight classifier call on the draft output — not the full pragmatic-analysis model — to keep the added latency small. It is logged to the same trace as the draft itself, so a blocked draft is exactly as auditable as a blocked analysis.

---

## Step 5 — Traceability

Each interaction generates a persistent record identified by a unique `trace_id` that enables reconstruction of the agent's entire decision chain.

The record includes:
- Input message and context
- Retrieved fragments with vector score, lexical score, RRF rank, and reranker score
- Reranker model identity and version
- Embedding model identity and version
- Exact prompt and version (`prompt_version`)
- Model version
- Confidence level
- Applied policy (if any)
- `grounded` field
- Per-stage and total latency, timestamp

Accessible in real-time via `GET /audit/{trace_id}`.

---

## Latency Budget

Per-stage shape of a typical request on the happy path — relative weight, not fixed numbers:

| Stage | Relative cost | Notes |
|-|-|-|
| Policy validation | Negligible | Pure Python, no I/O |
| Crisis classifier | Small | Runs in parallel with retrieval |
| Embedding | Small | Parallel branch, local Ollama call |
| Hybrid search (vector + lexical + RRF) | Small | Parallel branch |
| Reranking | Small–moderate | Cross-encoder call |
| Pragmatic analysis | Dominant | Structured output, largest prompt |
| Trace persistence | Negligible | Async write |

Because steps 2a and 2b overlap, the parallel block costs `max(crisis, retrieval)` rather than their sum.

> **Local hardware note.** Actual millisecond figures depend entirely on the machine running Ollama (GPU, VRAM, model size) rather than on a fixed cloud SLA. The dominant cost is still the main LLM call — latency optimization should target that path (prompt size, output schema complexity, semantic caching) before anything else.

### Measured on RTX 3050 (4GB VRAM)

Benchmarked directly with `ollama run llama3.2:3b --verbose`, not simulated:

| Metric | Value | Notes |
|-|-|-|
| Model load (cold) | ~9.3 s | One-time cost per cold start; the model stays resident in VRAM between requests (`OLLAMA_KEEP_ALIVE`), so a warm backend does not pay this on every call |
| Prompt processing rate | ~252 tokens/s | Reading the input — not the bottleneck |
| Generation rate | **~18 tokens/s** | The real bottleneck — every output token costs ~55 ms |

The structured `/analyze` response (short JSON: meaning, signals, alert level, recommendation, confidence) generates far fewer tokens than a free-form paragraph — roughly 150–250 tokens. At ~18 tokens/s that puts the main LLM call at **~8–14 s once the model is warm**, well above the ~1.6 s the same step took on GPT-4.1 in the cloud version. This is the direct cost of running on a 4GB consumer GPU instead of provisioned cloud compute, measured rather than assumed. End-to-end pipeline latency has not been benchmarked yet — the number above is per-LLM-call only, pending the full backend.

---

## Orchestration: LangGraph

The governance pipeline is orchestrated with **LangGraph**, which provides:
- Directed graph execution of the cascade steps
- Parallel node execution for the crisis/retrieval branch
- Short-circuit logic when a policy triggers
- State management across pipeline steps
- Clear separation between orchestration and business logic

LangChain is used selectively, only where it adds value (e.g., document loaders for the indexing pipeline).

---

## Continuous Evaluation and Prompt Versioning

A periodic job (via **GitHub Actions** or **Celery + Redis** background tasks) executes a set of test questions and publishes results to a local metrics store, exposed via `GET /metrics`:
- Grounded percentage
- Policy compliance
- Confidence distribution
- Latency percentiles per stage
- Cache hit rate

Each prompt change generates a version (`v1.1`, `v1.2`…) present in every response and in `/audit`, enabling data-driven justification for each change and before/after behavior comparison.

The evaluation set also serves as the **recalibration harness** for the confidence thresholds whenever the reranker or embedding model changes.

---

## Structured Output

The analysis produces a JSON response validated by Pydantic v2:

```json
{
  "meaning": "...",
  "signals": ["...", "..."],
  "alert_level": "MEDIUM",
  "recommendation": "...",
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
    "latency_ms": 11500,
    "model": "llama3.2:3b",
    "prompt_version": "v1.2",
    "embedding_model": "nomic-embed-text",
    "rerank_model": "cross-encoder/ms-marco-MiniLM-L-6-v2",
    "from_cache": false,
    "sentiment": { "label": "neutral", "score": 0.62 }
  }
}
```
