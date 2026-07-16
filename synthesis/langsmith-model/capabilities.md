# LangSmith Capabilities — Synthesis Opportunities

## 1. Tracing & Spans

### What LangSmith Does
- Captures hierarchical run traces for LLM applications
- Every LLM call, tool invocation, chain step, and retrieval becomes a "run" (span)
- Runs form parent-child trees: a chain run contains LLM runs, tool runs, retriever runs
- Each run records: inputs, outputs, start/end time, token counts, model, metadata, error state
- Traces propagate context via headers or environment variables across service boundaries
- Auto-instrumentation for LangChain; manual SDK instrumentation for any Python/JS code

### Synthesis Opportunity
Map Claude Code's tool dispatch pipeline to LangSmith-style traces. Each agent session
becomes a root trace; each tool call becomes a child run. MxM gate decisions become
metadata annotations on the run. NemoClaw sandbox operations become nested spans.
**Result: full observability of the governance pipeline, not just the agent actions.**

---

## 2. Evaluation Pipelines

### What LangSmith Does
- Dataset-driven evaluation: run a chain/agent against a curated dataset of input-output pairs
- Custom evaluator functions score each output (correctness, relevance, toxicity, etc.)
- Comparison experiments: run two versions side-by-side on the same dataset
- Regression detection: flag when a new version scores worse than baseline
- Results stored as structured experiment records with per-example scores

### Synthesis Opportunity
Apply evaluation patterns to MxM gate performance. Create datasets of known-good and
known-bad tool calls. Run gate scripts against these datasets. Score: did the gate
correctly block/allow? Track regression when gates are modified.
**Result: continuous validation of governance rules, not just one-off testing.**

---

## 3. Dataset Management

### What LangSmith Does
- First-class dataset objects with versioning
- Examples are input-output pairs with optional metadata
- Datasets can be created from: production traces, manual upload, CSV, synthetic generation
- Split management for train/test/validation
- Examples are immutable once added (append-only, no silent mutation)

### Synthesis Opportunity
MxM's enforcement.log is already append-only, but unstructured. LangSmith's dataset model
provides a pattern for structuring audit records as evaluable datasets:
- Each gate decision becomes a dataset example (input=tool call, output=allow/block)
- These can then feed evaluation pipelines for gate regression testing
**Result: audit log becomes a living test suite.**

---

## 4. Human-in-the-Loop Annotation

### What LangSmith Does
- Annotation queues: route production runs to human reviewers
- Configurable scoring rubrics (categorical, continuous, binary)
- Reviewer assignment and queue management
- Annotations stored alongside the run, enabling model fine-tuning and evaluation
- Supports multi-annotator agreement tracking

### Synthesis Opportunity
MxM's operator approval (permission mode "ask") is a simple y/n gate. LangSmith's
annotation model provides richer feedback: why was this approved? Confidence level?
Would you approve similar requests? This structured feedback could train better
gate heuristics over time.
**Result: operator decisions become training data for governance improvement.**

---

## 5. Prompt Versioning

### What LangSmith Does
- Prompts stored as versioned objects in a prompt hub
- Each version is immutable with a commit hash
- A/B testing: deploy two prompt versions and compare performance
- Deployment tracking: which prompt version is active in which environment
- Prompt templates support variable interpolation and structured messages

### Synthesis Opportunity
MxM governance instructions (CLAUDE.md, mission.md, morals.md, mind.md) are currently
versioned only via git. LangSmith's prompt versioning pattern enables:
- Treating governance documents as "prompts" with explicit versioning
- Comparing agent behavior across governance versions
- Rolling back governance changes if regression detected
**Result: governance-as-code with version control and regression testing.**

---

## 6. Feedback Scoring

### What LangSmith Does
- Feedback objects attached to any run: score, comment, source (human/auto/model)
- Aggregate scoring across runs for quality metrics
- Feedback drives filtering: find low-scoring runs for review
- Supports custom feedback keys (e.g., "helpfulness", "safety", "accuracy")
- Programmatic feedback via SDK alongside human feedback

### Synthesis Opportunity
Attach feedback scores to MxM gate decisions and agent actions. Auto-score based on
outcome (did the approved action succeed? did the blocked action get re-attempted?).
NemoClaw sandbox violations become automatic negative feedback.
**Result: quantitative quality metrics for the entire governance pipeline.**

---

## 7. Cost Attribution

### What LangSmith Does
- Token counts (input/output/total) per LLM run
- Model identification per run enables cost calculation
- Aggregation by trace, project, user, or time period
- Cost tracking across multi-model workflows (different models in same chain)

### Synthesis Opportunity
Claude Code spawns multiple agents; each makes multiple LLM calls. LangSmith's cost
model enables per-agent, per-task cost tracking. Combined with NemoClaw's resource
limits, this enables cost budgets: an agent gets both a sandbox resource limit AND
a token/cost budget.
**Result: resource governance covers both compute (NemoClaw) and inference (LangSmith).**

---

## Capability Matrix: What LangSmith Contributes

| Capability | LangSmith | Claude Code | NemoClaw | MxM | Synthesis Role |
|-----------|-----------|-------------|----------|-----|---------------|
| Hierarchical tracing | Primary | — | Container logs | enforcement.log | LangSmith pattern |
| Evaluation pipelines | Primary | — | — | — | LangSmith provides |
| Dataset management | Primary | — | — | Append-only log | LangSmith pattern |
| Human annotation | Primary | y/n approval | Operator TUI | — | LangSmith enriches |
| Prompt versioning | Primary | CLAUDE.md (git) | — | Git-versioned | LangSmith pattern |
| Feedback scoring | Primary | — | — | — | LangSmith provides |
| Cost attribution | Primary | — | cgroups | — | LangSmith provides |
| Runtime isolation | — | Process-level | Primary | — | NemoClaw provides |
| Semantic gates | — | Hooks | — | Primary | MxM provides |
| Agent management | — | Primary | — | — | Claude Code provides |
