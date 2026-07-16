# LangSmith Model

**Purpose:** Document LangSmith's public interfaces, observability architecture, and evaluation
patterns as a reference model for synthesis with MxM governance, NemoClaw sandboxing,
Claude Code agent management, and OpenCode agent loop.

**Legal basis:** This folder contains NO LangChain/LangSmith proprietary server code. All content is derived from:
- LangSmith Python/JS SDK (MIT license, open source on GitHub)
- Published documentation (docs.smith.langchain.com)
- Observable API behavior and public schemas
- LangChain ecosystem integration patterns (MIT-licensed)

Ideas, methods, interfaces, and behavioral descriptions are not copyrightable.
Only specific code expression is protected. We describe *what it does*, not *how the code does it*.

## Contents

- `capabilities.md` — Key capability areas and synthesis opportunities
- `interfaces.md` — SDK interface patterns, trace schema, evaluator contracts
- `observability-model.md` — Hierarchical tracing, spans, cost attribution, metadata
- `eval-model.md` — Evaluation architecture: datasets, evaluators, experiments, human feedback
- `gaps.md` — Capabilities absent from LangSmith that other models provide

## Relationship to Sibling Folders

```
internal-projects/
  nemoclaw/              # Apache 2.0 — NVIDIA's sandbox runtime (source)
  claude-code-model/     # Behavioral reference — Anthropic's agent management
  langsmith-model/       # Behavioral reference — LangChain's observability & eval
```

## Role in the Synthesis Landscape

LangSmith occupies a distinct niche: **observability and evaluation** for LLM applications.
Where Claude Code focuses on agent management and NemoClaw on runtime isolation,
LangSmith provides the instrumentation layer:

1. **Tracing** — Hierarchical run traces with parent-child spans, token counts, latency, cost
2. **Evaluation** — Dataset-driven evaluation pipelines with custom scoring and regression testing
3. **Human feedback** — Annotation queues, feedback scoring, and human-in-the-loop workflows
4. **Prompt management** — Versioned prompts with A/B comparison and deployment tracking

The synthesis target extracts these patterns for integration with:
- **MxM governance** — Audit logging gains structured trace format; gate decisions become spans
- **NemoClaw** — Sandbox operations become traceable runs with cost/resource attribution
- **Claude Code** — Agent tool calls become hierarchical traces with parent-child relationships
- **OpenCode** — Agent loop iterations become observable evaluation cycles
