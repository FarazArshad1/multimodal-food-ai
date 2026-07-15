# Self-Improving Agents

A hands-on project inspired by [DoorDash's AI-led metadata platform](architecture/doordash_architecture.md). The goal is **not** to build vision/multimodal analysis. Instead, you will learn and implement **reinforcement-learning-inspired prompt optimization** — a system where prompts improve themselves through an automated feedback loop.

## What You Are Building

A text-only pipeline that:

1. Ingests menu items (name + description)
2. Generates metadata tags using an LLM + prompt
3. Evaluates outputs with an LLM jury (consensus scoring)
4. Uses failure signals to automatically rewrite and improve the prompt
5. Repeats until metrics on a hold-out evaluation set stop improving

Think of it as **optimizing the prompt instead of model weights** — same loop dynamics as RL, but faster and cheaper.

```
┌─────────────┐     ┌──────────────┐     ┌─────────────┐     ┌──────────────────┐
│  Ingestion  │ ──▶ │  Generation  │ ──▶ │  LLM Jury   │ ──▶ │ Prompt Optimizer │
│  (Phase 1)  │     │  (Phase 2)   │     │  (Phase 3)  │     │    (Phase 4)     │
└─────────────┘     └──────────────┘     └─────────────┘     └────────┬─────────┘
       ▲                                                                │
       │                     failure cases + metrics                     │
       └────────────────────────────────────────────────────────────────┘
```

## Learning Goals

By the end of this project you should understand:

- How to structure an evaluation dataset that drives optimization (data quality = optimization direction)
- Why **failure cases carry more signal** than successes for prompt tuning
- How to build an LLM jury (multi-model consensus) as a reward function
- How to implement a tuning agent that reads failures and proposes targeted prompt changes
- How to guard against prompt regression using hold-out metrics (precision, recall, F1)

## What We Are Skipping (For Now)

| DoorDash feature | Why skipped in this learning project |
|---|---|
| Vision / image signals | Focus is prompt RL, not multimodal inference |
| Fine-tuned SLMs | Use off-the-shelf LLM APIs first |
| Spark / distributed inference | Local/small-batch processing is enough to learn the loop |
| Merchant overrides | Out of scope until core loop works |
| GEPA / population-based search | We use failure-signal-driven targeted edits (DoorDash's chosen approach) |

---

## Phases

### Phase 1 — Ingestion & Data Foundation `← YOU ARE HERE`

**Goal:** Load, validate, and deduplicate menu items. Split data into train/eval sets that the optimizer will use later.

**Deliverables:**
- [ ] Sample menu dataset (JSON/CSV) with items and ground-truth tags
- [ ] Ingestion module: load, validate schema, deduplicate by `(name, description)`
- [ ] Train / eval split (eval set is the optimizer's guardrail — never tune on it directly)
- [ ] CLI or script: `python -m src.ingest --input data/menu_items.json`

**Key concepts:**
- Deduplication avoids redundant LLM calls (same principle as DoorDash's exact-feature dedup)
- Evaluation data quality directly determines optimization direction — invest time here

**Suggested directory layout (created as we go):**
```
data/
  menu_items.json          # raw items + ground-truth tags
  menu_items.deduped.json  # after deduplication
src/
  ingest/
    loader.py              # read & validate
    deduplicator.py        # exact-match dedup
    splitter.py            # train/eval split
tests/
  test_ingest.py
```

---

### Phase 2 — Baseline Generation

**Goal:** Run a fixed starter prompt against ingested items and collect raw LLM outputs.

**Deliverables:**
- [ ] Prompt template (versioned, e.g. `prompts/v0_baseline.txt`)
- [ ] Generation agent: call LLM API, parse structured tag output
- [ ] Output store: `results/generation_run_<timestamp>.jsonl`
- [ ] Baseline metrics vs ground truth (rough precision/recall)

**Key concepts:**
- Start with a deliberately mediocre prompt — you need room to improve
- Version every prompt; the optimizer will fork from the latest best

---

### Phase 3 — LLM Jury Evaluation

**Goal:** Replace simple string-matching with a consensus evaluator that acts as the **reward function**.

**Deliverables:**
- [ ] Multiple evaluator prompts (2–3 "jurors" with different perspectives)
- [ ] Per-tag verdict aggregation (majority vote + rationale logging)
- [ ] Evaluation report: per-item failures with reasons
- [ ] Metrics: precision, recall, F1 on the hold-out eval set

**Key concepts:**
- The jury score **is** the reward signal for prompt optimization
- Tag-level verification (not whole-item pass/fail) gives finer-grained failure signals

---

### Phase 4 — RL-Inspired Prompt Optimization Loop

**Goal:** The core learning objective — an autonomous loop that improves prompts from failures.

**Deliverables:**
- [ ] Tuning agent: reads failure cases, proposes targeted prompt edits (rules, examples, constraints)
- [ ] Candidate prompt testing on eval set with metric guardrails
- [ ] Accept/reject logic: only promote prompt if eval metrics improve
- [ ] Iteration log: `runs/optimization_<timestamp>/` with prompts, metrics, failure analyses
- [ ] CLI: `python -m src.optimize --max-iterations 10`

**The loop (mirrors Figure 3 in the architecture doc):**
```
1. Score current prompt on eval set        → metrics (reward)
2. Collect failure cases (weighted heavily) → failure signals
3. Tuning agent proposes prompt v(N+1)     → candidate action
4. Score candidate on eval set             → new reward
5. Accept if metrics ↑, else reject & retry
6. Repeat until convergence or max iterations
```

**Key concepts:**
- Failure-weighted signals > success-only signals
- Prompt optimization has convergence dynamics similar to weight training
- Low-quality eval data causes the agent to chase noise — Phase 1 matters

---

### Phase 5 — Orchestration & Observability (Stretch)

**Goal:** Wire phases into a single runnable pipeline with logging and comparison across prompt versions.

**Deliverables:**
- [ ] End-to-end runner: ingest → generate → evaluate → optimize
- [ ] Prompt version registry with metric history
- [ ] Simple dashboard or markdown report per optimization run
- [ ] Config file for model choice, jury size, iteration limits

---

## Tech Stack (Suggested)

| Layer | Choice | Notes |
|---|---|---|
| Language | Python 3.11+ | Simple scripts, easy LLM SDKs |
| LLM API | OpenAI / Anthropic / local Ollama | Start with one provider |
| Data format | JSON / JSONL | Human-readable, git-friendly |
| Config | `.env` + `pyproject.toml` | API keys in `.env`, never committed |
| Tests | `pytest` | Unit tests for ingestion & metrics |

---

## Getting Started

### Prerequisites

- Python 3.11+
- An LLM API key (needed from Phase 2 onward; Phase 1 is API-free)

### Phase 1 — Next Steps

1. Create the sample dataset under `data/`
2. Implement the ingestion module under `src/ingest/`
3. Run deduplication and produce train/eval splits
4. Write tests to lock in schema validation and dedup behavior

When Phase 1 is done, you will have clean, deduplicated data and a reliable eval set — the foundation everything else depends on.

---

## Project Principles

1. **Text-only first** — learn the optimization loop before adding complexity
2. **Eval set is sacred** — never leak eval examples into prompt tuning
3. **Version everything** — prompts, runs, metrics
4. **Failures drive learning** — log them richly; they are the training signal
5. **Small steps** — complete and test each phase before moving on

---

## Reference

- [architecture/doordash_architecture.md](architecture/doordash_architecture.md) — original system design this project is based on
