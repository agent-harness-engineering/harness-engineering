# LangSmith Evaluation Model

Evaluation architecture derived from the MIT-licensed LangSmith SDK
and published documentation.

## Dataset-Driven Evaluation

### Core Pattern
LangSmith evaluation is built on a simple loop:

```
For each example in dataset:
    1. Run the target function on example.inputs
    2. Pass (run, example) to each evaluator
    3. Record scores as experiment results
```

```python
from langsmith import evaluate

results = evaluate(
    target=my_agent,                    # function to evaluate
    data="gate-decisions",              # dataset name
    evaluators=[correctness, latency],  # scoring functions
    experiment_prefix="v2.1",           # experiment label
)
```

### Key Properties
- **Deterministic dataset** — same inputs every run, enabling fair comparison
- **Pluggable target** — any callable that takes inputs and returns outputs
- **Pluggable evaluators** — any function matching the evaluator contract
- **Versioned experiments** — each evaluation run is a named, immutable record

### Synthesis Application: Gate Evaluation
```
Dataset: "gate-test-cases"
  Example 1: {input: {tool: "Bash", cmd: "rm -rf /"}, expected: "block"}
  Example 2: {input: {tool: "Read", path: "README.md"}, expected: "allow"}
  Example 3: {input: {tool: "Bash", cmd: "git push --force"}, expected: "block"}

Target: MxM gate script
Evaluators: [decision_correctness, latency_budget]
```

**Result: automated regression testing for governance rules.**

## Custom Evaluators

### Interface
```python
def evaluator(run: Run, example: Example) -> EvaluationResult:
    return {
        "key": "metric-name",
        "score": float | bool | int,
        "comment": "optional explanation",
    }
```

### Evaluator Categories
| Category | What It Measures | Example |
|----------|-----------------|---------|
| Correctness | Output matches reference | Exact match, fuzzy match, semantic similarity |
| Quality | Output meets quality bar | LLM-as-judge, rubric scoring |
| Safety | Output meets safety requirements | Toxicity, PII detection, policy compliance |
| Performance | Execution characteristics | Latency, token count, cost |
| Structural | Output format compliance | JSON validity, schema conformance |

### LLM-as-Judge Pattern
```python
def llm_judge(run: Run, example: Example) -> dict:
    prompt = f"""
    Given input: {run.inputs}
    Expected: {example.outputs}
    Actual: {run.outputs}
    Score correctness 0-1.
    """
    score = llm.invoke(prompt)  # uses a judge model
    return {"key": "llm-correctness", "score": float(score)}
```

### Synthesis Application: Governance Evaluators
| Evaluator | Tests | Applies To |
|-----------|-------|-----------|
| Gate correctness | Did gate block/allow correctly? | MxM gates |
| False positive rate | How often does gate block legitimate actions? | MxM gates |
| Latency budget | Does gate execute within time budget? | MxM hooks |
| Sandbox escape | Did sandboxed action stay within policy? | NemoClaw |
| Audit completeness | Was every decision logged? | MxM enforcement.log |
| Agent efficiency | Did agent complete task within token budget? | Claude Code agents |

## Regression Testing

### Pattern
```python
# Run baseline
baseline = evaluate(target=gate_v1, data="gate-tests", experiment_prefix="baseline")

# Run candidate
candidate = evaluate(target=gate_v2, data="gate-tests", experiment_prefix="candidate")

# Compare
comparison = client.compare_experiments(
    experiment_ids=[baseline.id, candidate.id]
)
# Shows per-example score deltas
```

### Regression Detection
- Per-example comparison: which specific cases got better or worse?
- Aggregate comparison: did overall accuracy change?
- Statistical significance: is the change real or noise?

### Synthesis Application
When modifying MxM gate scripts:
1. Run current gates against test dataset → baseline scores
2. Modify gate script
3. Run modified gates against same dataset → candidate scores
4. Compare: any regressions?
5. If regression detected: investigate specific examples that got worse
6. If no regression: deploy modified gate

**Result: governance changes are validated before deployment.**

## Comparison Experiments

### Side-by-Side Evaluation
```python
# Evaluate two different approaches on the same dataset
experiment_a = evaluate(target=approach_a, data="test-set")
experiment_b = evaluate(target=approach_b, data="test-set")

# Pairwise evaluator: which is better per-example?
def pairwise_judge(run_a: Run, run_b: Run, example: Example) -> dict:
    # Compare outputs of both runs
    return {
        "key": "preference",
        "score": 1 if prefer_a else 0,
    }
```

### Use Cases for Synthesis
| Comparison | A | B | Decision |
|-----------|---|---|----------|
| Gate implementation | Regex-based | LLM-based | Which has fewer false positives? |
| Agent strategy | Single agent | Multi-agent | Which is more cost-effective? |
| Sandbox policy | Permissive | Restrictive | Which balances safety and usability? |
| Governance version | v1 morals.md | v2 morals.md | Does update cause regressions? |

## Annotation Queues

### Purpose
Route production runs to human reviewers for quality assessment:

```
Production trace → Filter (error/low-score/random sample)
                     → Annotation queue
                       → Human reviewer
                         → Feedback (score + comment)
                           → Stored on run
```

### Queue Configuration
- **Filter criteria** — which runs enter the queue (errors, low auto-scores, random sample)
- **Rubric** — what reviewers score (categorical labels, 1-5 scales, binary pass/fail)
- **Assignment** — round-robin, expertise-based, or first-available
- **Throughput** — target review rate

### Synthesis Application: Operator Review Queues
MxM's operator approval is synchronous (blocks until approved). LangSmith's annotation
queue pattern enables asynchronous review:

| Current (MxM) | With Annotation Pattern |
|-------------|------------------------|
| Gate blocks → operator approves now | Gate blocks → action queued → operator reviews later |
| All blocked actions require approval | Sample of approved actions reviewed post-hoc |
| No feedback captured | Structured feedback on every reviewed action |
| No quality metrics | Reviewer agreement and gate accuracy tracked |

## Human Feedback Integration

### Feedback Loop
```
1. Agent acts → traced as run
2. Run scored by auto-evaluators
3. Low-scoring runs enter annotation queue
4. Human reviewer provides feedback
5. Feedback stored on run
6. Aggregated feedback → evaluator tuning
7. Improved evaluators → better auto-scoring
```

### Synthesis Design
```
1. Agent calls tool → traced with MxM gate decision
2. Gate decision auto-evaluated (correctness, latency)
3. Low-confidence decisions enter operator review queue
4. Operator provides structured feedback (not just y/n)
5. Feedback drives gate improvement:
   - New test cases added to evaluation dataset
   - Gate thresholds adjusted
   - False positive patterns documented
6. Improved gates deployed → fewer operator interrupts
```

**Result: governance that learns from operator feedback, not just static rules.**

## Evaluation Architecture Summary

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│   Datasets   │────▶│   Evaluate   │────▶│ Experiments  │
│ (test cases) │     │ (target +    │     │ (results +   │
│              │     │  evaluators) │     │  scores)     │
└──────┬───────┘     └──────────────┘     └──────┬───────┘
       │                                         │
       │  ◀── production traces                  │
       │       fed back as examples              │
       │                                         ▼
┌──────┴───────┐                          ┌──────────────┐
│  Annotation  │◀─────────────────────────│  Comparison  │
│   Queues     │   low-scoring runs       │  (baseline   │
│ (human       │   routed for review      │   vs new)    │
│  review)     │                          └──────────────┘
└──────┬───────┘
       │
       ▼
┌──────────────┐
│   Feedback   │──── drives dataset growth + evaluator tuning
│ (structured  │
│  scores)     │
└──────────────┘
```

In synthesis, this architecture governs governance itself:
- Gate decisions are the "target" under evaluation
- Test cases are known-good/known-bad tool calls
- Experiments compare gate versions
- Human feedback improves gate accuracy over time
