# LangSmith Observability Model

How LangSmith structures observability for LLM applications, derived from
the MIT-licensed SDK and published documentation.

## Hierarchical Run Traces

### Core Concept
Every operation in an LLM application becomes a **run** (span). Runs nest to form
trees that capture the full execution path:

```
Root: agent-session (chain)
├── prompt-load (prompt)
├── llm-call-1 (llm)
├── tool-search (tool)
│   ├── retriever-lookup (retriever)
│   └── llm-rerank (llm)
├── llm-call-2 (llm)
└── output-parse (parser)
```

### Properties
- **Trace ID** groups all runs in a single execution into one trace
- **Parent Run ID** establishes the hierarchy (null = root)
- **Dotted Order** is a hierarchical sort key encoding depth and sequence
- Traces can span service boundaries via header propagation

### Synthesis Mapping: Agent Session as Trace

```
Root: Claude-Code-session (chain)
├── claude-md-load (prompt)              ← CLAUDE.md injection
├── 4m-gate-check (tool)                 ← Morals/Mind gate
├── tool-dispatch: Read (tool)           ← Tool call
│   ├── pretooluse-hook (tool)           ← 4M PreToolUse
│   ├── execution (tool)                 ← Actual Read
│   └── posttooluse-hook (tool)          ← 4M PostToolUse
├── agent-spawn: explore (chain)         ← Subagent becomes child trace
│   ├── tool-dispatch: Grep (tool)
│   └── tool-dispatch: Glob (tool)
├── tool-dispatch: Bash (tool)
│   ├── pretooluse-hook (tool)
│   │   └── nemoclaw-policy-check (tool) ← Sandbox policy
│   ├── execution (tool)
│   └── posttooluse-hook (tool)
└── session-end (chain)
```

Every layer of the synthesis stack becomes visible in a single trace tree.

## Parent-Child Spans

### Relationship Types
| Relationship | Meaning | Example |
|-------------|---------|---------|
| Chain → LLM | Orchestrator invokes model | Agent makes inference call |
| Chain → Tool | Orchestrator invokes tool | Agent calls Bash |
| Chain → Chain | Orchestrator invokes sub-orchestrator | Agent spawns subagent |
| Tool → LLM | Tool internally calls model | Tool uses LLM for parsing |
| Chain → Retriever | Orchestrator fetches context | Agent queries memory |

### Span Lifecycle
```
PENDING → RUNNING → COMPLETED | ERROR
```

Each state transition is timestamped, enabling:
- **Duration** = end_time - start_time
- **Queue time** = start_time - parent_end_of_previous_child (if sequential)
- **Concurrency** = overlapping child spans

### Synthesis Value
Claude Code currently has no span concept. Tool calls are fire-and-forget from an
observability perspective. Adopting the parent-child span model enables:
- Identifying which tool calls are bottlenecks (latency attribution)
- Tracing governance decisions back to the tool call that triggered them
- Detecting anomalous patterns (e.g., agent spawning excessive subagents)

## Token Counting

### What LangSmith Tracks
Per LLM run:
```python
{
    "token_usage": {
        "prompt_tokens": 1500,
        "completion_tokens": 350,
        "total_tokens": 1850,
    },
    "model": "claude-sonnet-4-20250514",
}
```

### Aggregation Levels
| Level | Scope | Use Case |
|-------|-------|----------|
| Run | Single LLM call | Cost per inference |
| Trace | All LLM calls in a session | Cost per task |
| Project | All traces in a project | Cost per workflow |
| Time period | All traces in a window | Budget tracking |

### Synthesis Value
Claude Code agents make multiple LLM calls per session but provide no per-agent
cost breakdown. Token counting at the trace level enables:
- Per-agent cost budgets (complementing NemoClaw's compute budgets)
- Cost comparison between agent strategies
- Anomaly detection: agent consuming 10x normal tokens indicates a loop

## Latency Tracking

### What LangSmith Tracks
```python
{
    "start_time": "2024-01-01T00:00:00.000Z",
    "end_time": "2024-01-01T00:00:02.350Z",
    # Derived:
    "latency": 2.35,  # seconds
    "first_token_latency": 0.45,  # for streaming LLM runs
}
```

### Latency Breakdown
For a trace with multiple spans:
```
Total trace latency: 15.2s
├── LLM calls: 8.3s (55%)
├── Tool executions: 4.1s (27%)
├── Hook evaluations: 1.8s (12%)
└── Overhead: 1.0s (6%)
```

### Synthesis Value
4M gates add latency to every tool call (hook scripts execute synchronously).
Latency tracking per span category reveals:
- How much overhead governance adds
- Which gates are slow (optimization targets)
- Whether NemoClaw's sandbox setup time is acceptable

## Cost Attribution Per Agent

### Pattern
LangSmith associates costs with traces, which map to logical units of work.
In a multi-agent system, each agent's trace is a separate cost center:

```
Project: ThinxS-session-2024-01-15
├── Agent: main (root)     — $0.045 (3 LLM calls)
├── Agent: explore-1       — $0.012 (1 LLM call, read-only)
├── Agent: explore-2       — $0.008 (1 LLM call, read-only)
└── Agent: general-1       — $0.032 (2 LLM calls, full tools)
                    Total: — $0.097
```

### Synthesis Design
Combine LangSmith's inference cost tracking with NemoClaw's compute resource tracking:

| Resource | Tracker | Budget Mechanism |
|----------|---------|-----------------|
| Tokens (in/out) | LangSmith pattern | Per-agent token limit |
| API cost ($) | LangSmith pattern | Per-task cost ceiling |
| CPU time | NemoClaw cgroups | Container CPU limit |
| Memory | NemoClaw cgroups | Container memory limit |
| Network bytes | NemoClaw policy | Egress byte limit |

**Result: unified resource governance spanning both inference and compute.**

## Metadata Propagation

### How LangSmith Propagates Context
```python
@traceable(metadata={"agent_id": "Agent_20240115_0930", "mode": "admin"})
def my_function():
    # metadata propagates to all child runs
    pass
```

Metadata flows:
1. **Downward** — parent metadata inherited by children (unless overridden)
2. **Queryable** — filter runs by metadata fields
3. **Indexable** — metadata keys become filterable dimensions

### Key Metadata Fields for Synthesis
| Field | Source | Purpose |
|-------|--------|---------|
| `agent_id` | Claude Code / 4M | Trace ownership |
| `session_id` | Claude Code | Session grouping |
| `permission_mode` | Claude Code | Security context |
| `sandbox_policy` | NemoClaw | Isolation level |
| `gate_decisions` | 4M | Governance trace |
| `governance_version` | 4M (git SHA) | Governance document version |
| `model_id` | Runtime | Model attribution |

### Synthesis Value
4M's enforcement.log captures gate decisions but without structured propagation.
LangSmith's metadata model enables:
- Correlating gate decisions with the agent and task that triggered them
- Filtering traces by governance version (did a rule change cause regressions?)
- Joining sandbox policy with inference traces for unified analysis

## Observability Architecture Summary

```
                    LangSmith Pattern
                    ┌─────────────────┐
                    │   Trace Store    │
                    │  (runs + spans)  │
                    └────────┬────────┘
                             │
              ┌──────────────┼──────────────┐
              │              │              │
         ┌────┴────┐   ┌────┴────┐   ┌────┴────┐
         │ Metrics  │   │  Query  │   │ Alerts  │
         │ (tokens, │   │ (filter,│   │ (cost,  │
         │  latency,│   │  search,│   │  error, │
         │  cost)   │   │  slice) │   │  latency│
         └─────────┘   └─────────┘   │  spike) │
                                     └─────────┘
```

In synthesis, the trace store replaces (or enriches) 4M's enforcement.log:
- Structured spans instead of flat log lines
- Queryable dimensions instead of grep
- Aggregatable metrics instead of manual counting
- Alert-capable instead of passive recording
