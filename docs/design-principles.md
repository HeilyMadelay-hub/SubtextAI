# Design Principles

## Mandatory Grounding

No response is delivered without documentary evidence. If confidence falls below threshold, generation is blocked before invoking the model.

The confidence gate reads the **top cross-encoder reranker score** — not raw cosine similarity, and not the RRF fusion score:

| Level | Top reranker score | Behavior |
|-|-|-|
| **HIGH** | ≥ 0.70 | Full response with fragment and section citation. `grounded=true`. |
| **MEDIUM** | 0.45 – 0.69 | Response with partial context warning. Available source is cited. |
| **LOW** | < 0.45 | System does not generate a response. |

Raw similarity is not calibrated across queries or embedding models, and RRF produces rank-based values on a different scale entirely. Only the cross-encoder score is bounded and comparable enough to gate on. See [AI Pipeline](ai-pipeline.md) for the full rationale.

---

## Full Traceability via `trace_id`

Every response is reconstructible: which documents influenced it, with what score, which prompt version, which model, and which policies were evaluated.

The persistent record includes:
- Input message and context
- Retrieved fragments with vector score, lexical score, RRF rank, and reranker score
- Reranker and embedding model identity and version
- Exact prompt and version (`prompt_version`)
- Model version
- Confidence level
- Applied policy (if any)
- `grounded` field
- Per-stage and total latency, timestamp

Accessible in real-time via `GET /audit/{trace_id}`.

---

## Agent Governance in Code

Security rules live in the pipeline, not in the prompt — the model never executes if a critical policy is violated.

Policies are hard rules coded in the FastAPI pipeline and orchestrated with LangGraph. Security behavior does not depend on the model or its configuration. If any policy triggers at any pipeline step, the flow stops and the main model is never invoked.

Crisis detection runs in parallel with retrieval for latency reasons, but remains strictly blocking: parallelism changes when the check happens, never whether it applies.

---

## Conversational Telemetry Engine

SubtextAI can be understood as a **telemetry engine applied to human conversation**. Just as mechanical system telemetry records and formalizes continuous signals — velocity, acceleration, friction — to describe behavior, SubtextAI formalizes dialogue dynamics from signals it already computes and persists per message (emotional intensity, intent changes, explicit vs. implicit meaning, tension).

These metrics are not a speculative layer or a new model: they are a formalization of data that already exists in the traces. They derive directly from persisted traces and inherit, without exception, the same grounding and traceability by `trace_id` as the rest of the system.

| Metric | Definition |
|-|-|
| **Emotional velocity** | Variation of emotional intensity between two consecutive messages (first derivative). |
| **Conflict acceleration** | Variation of emotional velocity (second derivative). Whether escalation is accelerating or decelerating. |
| **Conversational friction** | Accumulated tension between explicit and implicit meaning throughout the dialogue. |
| **Intent curves** | Trajectory of intent changes across the conversation. |

These are derivatives over model-assigned values, not physical measurements. Computing a second derivative over an uncertain signal produces a more precise number, not a more certain one. The metrics are designed to make patterns visible, not to assert objective magnitude.

---

## Swappable Document Corpus

The document corpus is swappable and configurable: the retrieval architecture is not coupled to any source, so any compatible document collection can be indexed without touching governance logic.

---

## Corpus Shapes Interpretation

The quality and perspective of the indexed document corpus directly shapes every interpretation the system produces. The same message analyzed against different corpora may produce different insights. Corpus selection is a product decision, not just a technical one.

---

## Descriptive System, Not Predictive

Current system capabilities are strictly descriptive, not predictive. Any predictive capability (such as estimating the impact of a message on subsequent turns) would require an independently trained and validated projection model, with its own evaluation and audit process, before it could be exposed as a reliable prediction.

---

## Confidence Is About Evidence, Not Truth

The confidence level reflects the quality of documentary grounding — how relevant the retrieved sources are — not the objective accuracy of the interpretation itself. A HIGH confidence means strong documentary context was found, not that the reading of intent is guaranteed to be correct.

This distinction is important: the system measures how well-supported an interpretation is, not how true it is. Users and developers should treat confidence as a measure of evidence strength, not as a certainty score.

---

## Traceability Under Data Protection

Full traceability and the right to erasure pull in opposite directions, and the tension is resolved by design rather than left implicit.

The system processes **private conversations**, which under GDPR may constitute special-category personal data when they reveal emotional health, relationships, or similar. The audit design — every analyzed message persisted immutably, forever — cannot coexist with Article 17 without an explicit resolution.

**How the conflict is resolved:**

| Mechanism | Design decision |
|-|-|
| **Split storage** | Raw message text and derived annotations are stored in separate columns. Annotations, scores, policy decisions, and model versions are the audit record; the raw text is the personal data. |
| **Text-level erasure** | An erasure request nulls the raw text while preserving the annotation layer. The audit trail stays intact and reconstructible; Replay Mode continues to work with the text redacted. |
| **Retention TTL** | Raw message text carries a configurable TTL (default 90 days) after which it is purged automatically. Annotations persist. |
| **Minimized third-party exposure** | Every model call — crisis classification, embeddings, pragmatic analysis — is transmitted to OpenRouter, which forwards it to the underlying model provider. This is a real data-residency and training-data question, not an avoided one: it must be resolved per provider (see [Cloud Deployment — Data Residency](cloud-deployment-aws.md#data-residency-and-compliance-notes)), not assumed away. What *is* still minimized: raw text never touches long-term storage outside the pipeline's own retention TTL, and no message content is logged by any AWS service beyond what the pipeline itself writes to the trace record. |

The design goal is that an audit remains fully reconstructible at the level of *decisions* even after the *content* that triggered them has been erased.

> **This is a real trade-off, not a footnote.** The earlier local-only design guaranteed zero third-party transmission by construction — no message ever left the machine. Moving generation and embeddings to OpenRouter trades that guarantee for a contractual one: no-training and data-handling commitments that vary by the specific model in use and must be verified, not assumed, before processing conversations that qualify as special-category data under GDPR.

---

## Governance Covers Generated Content, Not Just Retrieval

Every policy described so far validates the **input** — the message the system receives. Reply drafting (`/draft`) is different: it produces text the system puts in the user's mouth and sends to a third party under the user's name. Validating the request that asked for a draft and not the draft itself would leave the most consequential output in the system unchecked.

A dedicated policy, `harmful_draft_detected`, runs on the *generated* text before it is returned, catching manipulative or coercive phrasing that a well-formed request could still produce. See [AI Pipeline](ai-pipeline.md#step-5--output-validation) and [Testing Strategy](testing-strategy.md#draft-output-validation) for the implementation and how its accuracy is measured.

---

## Crisis Detection Is a Referral, Not a Response

When `crisis_detected` fires, the system does not attempt to counsel, calm, or engage with the user. It stops, and hands off to a concrete, configured support resource — never a generic "please seek help" message with nothing behind it.

Two things follow from that:

- **The resource must be real and region-appropriate.** A placeholder crisis line is worse than an obvious error, because it looks like help. Deploying to a new region without configuring a locally valid resource is a launch blocker, not a follow-up task.
- **Recall matters more than precision here.** Between over-triggering (an unnecessary block) and under-triggering (a missed crisis), the classifier is deliberately tuned toward the former. This trade-off is measured explicitly — see [Testing Strategy](testing-strategy.md#crisis-classifier-recall-over-precision) — rather than left to whatever a general-purpose prompt happens to produce.

---

## Informed Use, Not Blind Trust

SubtextAI interprets private conversations about relationships, work, and conflict — a domain where a wrong or over-confident answer can shape a real decision. The system is built to make that risk visible rather than hide it behind a confident tone:

- Every response distinguishes evidence strength (confidence level) from correctness (see [Confidence Is About Evidence, Not Truth](#confidence-is-about-evidence-not-truth) above).
- The product surface states plainly, at first use, that the tool does not replace professional advice, therapy, or legal counsel.
- Crisis handling is a referral to a human resource, never an attempt by the system to substitute for one.

None of this is a legal disclaimer bolted on afterward — it follows from the same principle as mandatory grounding: the system should never present more certainty than it actually has.
