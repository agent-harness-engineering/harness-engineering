# LangSmith Public Interfaces

All information below is derived from the MIT-licensed LangSmith SDK and published documentation.

## SDK Client Initialization

### Python
```python
from langsmith import Client

client = Client(
    api_url="https://api.smith.langchain.com",  # or self-hosted
    api_key="ls-...",
)
```

### JavaScript/TypeScript
```typescript
import { Client } from "langsmith";

const client = new Client({
  apiUrl: "https://api.smith.langchain.com",
  apiKey: "ls-...",
});
```

Environment variable fallbacks: `LANGCHAIN_API_KEY`, `LANGCHAIN_ENDPOINT`, `LANGCHAIN_PROJECT`.

## Trace / Run Schema

The core data object is a **Run** (equivalent to a span in OpenTelemetry):

```python
{
    "id": "uuid",
    "name": "ChatOpenAI",           # component name
    "run_type": "llm",              # llm | chain | tool | retriever | prompt | parser
    "parent_run_id": "uuid | null", # null = root trace
    "trace_id": "uuid",            # groups all runs in a trace
    "dotted_order": "20240101T...", # hierarchical sort key
    "inputs": {...},                # serialized input
    "outputs": {...},               # serialized output
    "error": "string | null",
    "start_time": "ISO-8601",
    "end_time": "ISO-8601",
    "extra": {                      # metadata bag
        "metadata": {...},
        "runtime": {...},
    },
    "events": [...],                # structured events within the run
    "tags": ["tag1", "tag2"],
    "session_name": "project-name", # project grouping
    "reference_example_id": "uuid | null",  # if from eval dataset
    "feedback_stats": {...},        # aggregated feedback
}
```

### Run Types and Their Semantics
| Type | Represents | Typical Parent |
|------|-----------|----------------|
| `chain` | Orchestration step (agent, chain, graph) | Root or chain |
| `llm` | Model invocation | chain |
| `tool` | Tool/function execution | chain |
| `retriever` | Vector/document retrieval | chain |
| `prompt` | Prompt template rendering | chain |
| `parser` | Output parsing | chain |

### Synthesis Mapping
| LangSmith Run Type | Claude Code Equivalent | 4M Equivalent |
|--------------------|-----------------------|---------------|
| `chain` (root) | Agent session | Session log entry |
| `chain` (nested) | Subagent spawn | — |
| `llm` | Model inference call | — |
| `tool` | Tool dispatch (Read, Bash, etc.) | Gate decision target |
| `retriever` | — | Memory recall |
| `prompt` | CLAUDE.md injection | Governance document load |

## Run Tree (Hierarchical Tracing)

The SDK provides a `RunTree` object for manual instrumentation:

```python
from langsmith.run_trees import RunTree

# Create root
root = RunTree(
    name="my-agent",
    run_type="chain",
    inputs={"query": "..."},
)

# Create child
child = root.create_child(
    name="tool-call",
    run_type="tool",
    inputs={"tool": "search", "query": "..."},
)
child.end(outputs={"result": "..."})
child.post()

root.end(outputs={"answer": "..."})
root.post()
```

Key properties:
- `create_child()` sets `parent_run_id` and `trace_id` automatically
- `dotted_order` encodes the full hierarchy as a sortable string
- Posting is explicit — runs are buffered and batched by default

### Context Manager Pattern
```python
from langsmith import traceable

@traceable(run_type="chain", name="my-agent")
def my_agent(query: str):
    # All nested @traceable calls become children
    result = my_tool(query)
    return result

@traceable(run_type="tool")
def my_tool(query: str):
    return {"result": "..."}
```

The `@traceable` decorator handles run creation, context propagation, and posting.

## Evaluator Contract

Evaluators are functions conforming to this interface:

```python
from langsmith.schemas import Example, Run

def my_evaluator(run: Run, example: Example) -> dict:
    """
    Args:
        run: The completed run with inputs/outputs
        example: The dataset example with input/expected output
    Returns:
        {"key": "metric-name", "score": float_or_bool, "comment": "optional"}
    """
    prediction = run.outputs["answer"]
    reference = example.outputs["answer"]
    return {
        "key": "correctness",
        "score": prediction == reference,
    }
```

### Evaluator Types
| Type | Input | Use Case |
|------|-------|----------|
| Run-level | Single run + example | Per-example scoring |
| Summary | All runs in experiment | Aggregate metrics |
| Pairwise | Two runs on same example | A/B comparison |

### Synthesis Mapping
4M gate scripts follow a similar pattern: input (tool call context) → decision (allow/block).
The evaluator contract could formalize gate testing:
```
gate_evaluator(run=tool_call_context, example=known_good_decision) → score
```

## Feedback API

```python
# Create feedback on a run
client.create_feedback(
    run_id="uuid",
    key="safety",
    score=1.0,                    # numeric score
    value="safe",                 # categorical value
    comment="No policy violations",
    source_info={"type": "auto"}, # human | auto | model
)

# Query feedback
feedbacks = client.list_feedback(run_ids=["uuid"])
```

### Feedback Schema
```python
{
    "id": "uuid",
    "run_id": "uuid",
    "key": "metric-name",
    "score": 1.0,           # float, optional
    "value": "category",    # string, optional
    "comment": "text",
    "correction": {...},    # suggested correct output
    "source_info": {
        "type": "human | auto | model",
        "metadata": {...},
    },
    "created_at": "ISO-8601",
}
```

## Dataset API

```python
# Create dataset
dataset = client.create_dataset(
    dataset_name="gate-decisions",
    description="Known-good gate allow/block decisions",
)

# Add examples
client.create_example(
    inputs={"tool": "Bash", "command": "git push --force origin main"},
    outputs={"decision": "block", "reason": "force push to main"},
    dataset_id=dataset.id,
    metadata={"gate": "force-push", "severity": "high"},
)

# List examples
examples = client.list_examples(dataset_id=dataset.id)
```

### Dataset Schema
```python
{
    "id": "uuid",
    "name": "dataset-name",
    "description": "...",
    "created_at": "ISO-8601",
    "modified_at": "ISO-8601",
    "example_count": 42,
    "data_type": "kv",      # kv | llm | chat
}
```

### Example Schema
```python
{
    "id": "uuid",
    "dataset_id": "uuid",
    "inputs": {...},
    "outputs": {...},        # expected/reference output
    "metadata": {...},
    "created_at": "ISO-8601",
    "modified_at": "ISO-8601",
    "source_run_id": "uuid | null",  # if created from production trace
}
```

## Project / Session Grouping

Runs are organized into **projects** (historically called "sessions"):

```python
# Set project via env
os.environ["LANGCHAIN_PROJECT"] = "my-project"

# Or via SDK
client.create_project(project_name="my-project")
runs = client.list_runs(project_name="my-project", filter="eq(status, 'error')")
```

### Run Filtering (Query Language)
```python
# Filter by status
runs = client.list_runs(filter="eq(status, 'error')")

# Filter by latency
runs = client.list_runs(filter="gt(latency, 5.0)")

# Filter by feedback score
runs = client.list_runs(filter="lt(feedback_score, 0.5)")

# Filter by metadata
runs = client.list_runs(filter="has(metadata, 'agent_id')")

# Compound filters
runs = client.list_runs(
    filter="and(eq(run_type, 'tool'), gt(latency, 2.0))"
)
```

## Annotation Queue API

```python
# Create queue
queue = client.create_annotation_queue(
    name="safety-review",
    description="Runs flagged for safety review",
)

# Add runs to queue
client.add_runs_to_annotation_queue(
    queue_id=queue.id,
    run_ids=["uuid1", "uuid2"],
)

# Rubric: scoring criteria for reviewers
# Configured per-queue with categorical or continuous scales
```

## Key Integration Points for Synthesis

| Interface | Pattern Extracted | Synthesis Target |
|-----------|------------------|-----------------|
| `@traceable` decorator | Zero-config instrumentation | Wrap Claude Code tool dispatch |
| Run tree hierarchy | Parent-child span relationships | Agent-subagent trace hierarchy |
| Evaluator contract | `(run, example) → score` | Gate regression testing |
| Feedback API | Multi-source scoring on runs | Governance quality metrics |
| Dataset API | Versioned input-output pairs | Audit log as test suite |
| Annotation queues | Human review routing | Operator approval enrichment |
| Run filtering | Structured query over traces | Governance analytics |
