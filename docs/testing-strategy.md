# Testing Strategy

The system's central claim is that governance actually governs — that policies fire when they should, that grounding is real, and that the crisis classifier catches what it needs to catch. None of that is credible without a way to verify it. This document describes how each governed behavior is tested, as distinct from the [Continuous Evaluation](ai-pipeline.md#continuous-evaluation-and-prompt-versioning) job, which monitors production rather than gating changes before they ship.

---

## Test Layers

| Layer | Tool | What it covers |
|-|-|-|
| **Unit** | Pytest | Policy Engine logic, RRF fusion math, confidence gate thresholds — pure functions, no LLM calls |
| **Integration** | Pytest + test database | Full pipeline against a seeded PostgreSQL/pgvector instance, mocked LLM responses |
| **Evaluation** | Custom harness | Golden-set accuracy against real model calls — crisis classifier recall, grounding precision, draft safety |
| **Adversarial** | Custom harness | Prompt injection and jailbreak attempts against the policy layer |
| **Frontend** | Vitest + Testing Library | Component behavior, Replay Mode reconstruction from fixture traces |

Unit and integration tests run on every pull request via GitHub Actions and must pass before merge. Evaluation and adversarial suites run on the same schedule as the continuous evaluation job, since both call the live model and cost tokens.

---

## Crisis Classifier: Recall Over Precision

The crisis classifier is the one component where a wrong answer has a materially different cost depending on its direction. A false positive costs the user an unnecessary block and a redirect. A **false negative** — missing a real crisis — costs a person the intervention they needed. The evaluation harness is built around that asymmetry.

**Golden set composition:**
- Confirmed crisis-signal messages (sourced from clinical literature and crisis-line training material, not real user data)
- Adjacent-but-not-crisis messages (sadness, frustration, conflict) — designed to probe the boundary and catch over-triggering
- Clean messages across all supported contexts — the negative class

**Primary metric: recall on the crisis class.** Precision is tracked but is explicitly the secondary metric — the classifier is tuned to over-trigger rather than under-trigger. A false positive is a UX cost; a false negative is a safety failure. This trade-off is a deliberate design decision, not a tuning oversight, and it is documented here so it isn't relitigated silently in a future prompt change.

**Process:** every `crisis_classifier` prompt version is scored against the full golden set before it can replace the current `prompt_version` in production. A regression in recall blocks the promotion regardless of any precision gain.

---

## Grounding and Confidence Gate

The confidence gate (see [AI Pipeline](ai-pipeline.md#step-3--reranking-and-confidence-gate)) is tested for two distinct properties:

1. **Calibration** — does a HIGH-confidence response actually correspond to a chunk a human would judge as genuinely relevant? Measured against a labeled relevance set: query/document pairs scored by a human rater, compared against the reranker's output.
2. **Threshold correctness** — do the `0.70` / `0.45` cutoffs behave as intended on the current reranker model? This is re-run — not assumed — every time the reranker or embedding model changes, using the same evaluation set referenced in [AI Pipeline](ai-pipeline.md#continuous-evaluation-and-prompt-versioning).

A gate that never blocks anything is exactly as broken as one that blocks everything; the evaluation harness checks the block rate on a set of intentionally weak queries (out-of-corpus questions) to confirm `mandatory_grounding` actually activates.

---

## Draft Output Validation

The `harmful_draft_detected` policy (see [AI Pipeline](ai-pipeline.md#step-5--output-validation)) is tested against a set of prompts constructed to produce borderline output: requests where an "assertive" tone could plausibly tip into coercive language.

Metric: **catch rate** on a labeled set of known-manipulative reference replies, plus a **false-block rate** on a labeled set of legitimately firm-but-fair replies. Both are tracked — this is the one policy in the system where over-blocking has a direct product cost (a broken feature) as well as an under-blocking cost (a harm delivered under the product's name), so neither side is allowed to move without the other being checked.

---

## Adversarial Testing

The policy layer (`prompt_injection`, `unsupported_language`, `minimum_message`, `harmful_draft_detected`) is exercised against a maintained set of known jailbreak and injection patterns, updated as new techniques are identified. This suite is intentionally separate from the crisis and grounding golden sets because it targets a different failure mode: not "is the model right", but "can the model be talked into ignoring the rules that sit in front of it."

A pipeline where policies live in code rather than in the prompt (see [Design Principles](design-principles.md#agent-governance-in-code)) is exactly what makes this suite meaningful — an injection that fools the LLM should still be caught by `PolicyEngine`, which never asks the model's opinion of its own compliance.

---

## What This Does Not Cover

The evaluation harness measures the system against its own golden sets — it does not certify clinical accuracy or replace review by a mental health professional. Reaching a target recall on the crisis classifier's evaluation set is a statement about that set, not a guarantee about every message the system will ever see. This limitation is stated here explicitly and referenced from [Design Principles](design-principles.md) rather than left implicit.
