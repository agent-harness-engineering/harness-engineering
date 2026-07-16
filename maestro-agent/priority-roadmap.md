# Maestro Agent Implementation Priority Roadmap

This roadmap sequences the Maestro Agent build based on the [gap analysis](gap-analysis.md). Priority levels reflect safety criticality: P0 components are required before any agentic execution is safe; P1 components are required for production-quality operation; P2 and P3 improve robustness and extensibility.

## Priority Levels

| Level | Criterion | Implication |
|-------|-----------|-------------|
| **P0** | Safety-critical | Must be complete before any agent executes against real systems |
| **P1** | Production-quality | Required for reliable multi-model operation |
| **P2** | Robustness | Improves isolation and developer experience |
| **P3** | Extensibility | Enables community and plugin ecosystem |

---

## P0: Safety Foundation

These components implement the Morals module's executable enforcement requirement. Without them, the Maestro Agent is an uncontrolled execution path.

### 1. Tool Interception Proxy

**Goal**: No tool call reaches execution without passing through a normalization and validation layer.

**Deliverables**:
- Canonical tool call schema (model-agnostic internal representation)
- Parser: OpenAI function calling format
- Parser: Anthropic tool_use format
- Parser: Raw text action tag extraction (regex)
- Parameter schema validation against tool definitions
- Result formatter: convert canonical results back to model-expected format

**Acceptance criteria**:
- All three parser paths produce identical canonical output for equivalent tool calls
- Malformed tool calls are rejected with structured error, never forwarded to execution
- 100% of tool executions are logged with canonical representation

**MxM mapping**: Means (execution interface) + Morals (validation gate)

### 2. Permission Confirmation Channel

**Goal**: Every tool call not on the auto-allow list requires explicit human approval. Fail-closed on timeout.

**Deliverables**:
- Permission policy engine: auto-allow rules, deny rules, require-confirmation rules
- CLI relay (local development)
- Web UI relay (remote/headless operation)
- Webhook relay (CI/CD integration)
- Configurable timeout with fail-closed default (deny on no response)
- Approval audit log: timestamp, channel, approver, action context, decision

**Acceptance criteria**:
- Unapproved tool calls never execute
- Timeout produces deny, not hang
- Approval decisions are immutably logged
- Policy rules are testable in isolation

**MxM mapping**: Morals (deontic enforcement)

### 3. Self-Exemption Prevention

**Goal**: The agent process cannot modify its own governance configuration.

**Deliverables**:
- Read-only mount for gate scripts and policy configuration
- Separate process/credential boundary for governance changes
- Audit log for governance config access attempts
- Container deployment spec with appropriate bind mounts
- Integration test: agent attempts to modify config, attempt is blocked and logged

**Acceptance criteria**:
- No code path exists from agent process to governance config write
- Attempted access is logged and denied
- Configuration changes require out-of-band human action

**MxM mapping**: Morals (self-exemption prevention, defense in depth)

---

## P1: Production Quality

These components enable reliable multi-model operation and output quality assurance.

### 4. Context Window Management

**Goal**: Per-model token awareness with automatic compaction before overflow.

**Deliverables**:
- Tokeniser registry: tiktoken (GPT), sentencepiece (Gemini), Anthropic tokeniser, model-specific for open-weight
- Three-zone budget allocator (pinned / summary / verbatim) per MxM Memory spec
- Pre-compaction trigger at configurable threshold (default: 80% of context limit)
- Observation masking for older tool results
- Running summary generator for evicted turns
- Token budget dashboard/logging

**Acceptance criteria**:
- No context overflow errors in multi-turn sessions
- System prompt and persistent memory are never evicted
- Compaction is transparent to the user
- Token counts are accurate to within 5% of model's actual tokenization

**MxM mapping**: Memory (session continuity) + Mind (reasoning quality)

### 5. Structured Tool Output Parsing

**Goal**: Reliable extraction of tool calls from any supported model's output format.

**Deliverables**:
- Parser registry with format auto-detection
- OpenAI format parser with JSON argument extraction
- Anthropic format parser with content block traversal
- Raw text parser with configurable action tag patterns
- JSON schema validation on extracted parameters
- Consistent error format across all parsers

**Acceptance criteria**:
- Format auto-detection is correct for all supported models
- Invalid JSON in tool arguments is caught before execution
- Parser errors produce actionable diagnostics

**MxM mapping**: Means (tool interface) + Mind (tool-use planning)

### 6. Disposition Output Validation

**Goal**: Post-generation checks ensure model output meets mission-specific quality requirements.

**Deliverables**:
- Validation rule engine: confidence tags, citation format, JSON schema, content policy
- Per-mission validation profiles (different rules for coding vs research vs creative tasks)
- Retry loop: re-prompt with specific validation error (max 3 retries)
- Escalation path: after max retries, notify human rather than producing invalid output
- Validation metrics: pass rate, retry rate, escalation rate per model per mission

**Acceptance criteria**:
- Structured output always conforms to declared schema or is escalated
- Validation rules are declarative and testable
- Retry prompts include the specific failure reason

**MxM mapping**: Morals (output validation) + Mind (metacognition)

---

## P2: Robustness

These components harden the orchestrator for multi-tenant and production deployment.

### 7. Agent Isolation

**Goal**: Defense-in-depth containment of agent processes.

**Deliverables**:
- Filesystem sandbox: directory allowlist with path validation on all I/O
- Process isolation: separate OS user or container with restricted capabilities
- Resource limits: CPU, memory, network quotas via cgroups or container limits
- Network policy: egress restricted to approved endpoints
- Temporal limits: max execution time per tool call and per session
- Isolation test suite: escape attempt scenarios

**Acceptance criteria**:
- Agent cannot read or write outside sandbox
- Resource exhaustion in agent does not affect orchestrator
- Network egress to unapproved endpoints is blocked

**MxM mapping**: Morals (executable enforcement) + Means (execution boundary)

### 8. Model-Agnostic Streaming

**Goal**: Unified streaming interface regardless of LLM backend.

**Deliverables**:
- Streaming adapter: OpenAI SSE, Anthropic SSE, Google SSE, generic NDJSON
- Normalized internal event stream (text delta, tool call start, tool call delta, tool call end, error)
- Tool call buffering: detect partial JSON in stream, buffer until complete
- Backpressure handling: slow consumers don't crash the pipeline
- Streaming metrics: latency to first token, tokens per second per model

**Acceptance criteria**:
- UI layer consumes a single event format regardless of backend
- Switching models requires zero UI changes
- Partial tool call JSON is never forwarded to the parser

**MxM mapping**: Means (execution interface)

---

## P3: Extensibility

### 9. Skill/Plugin System

**Goal**: Third-party and internal plugins extend the orchestrator without modifying core code.

**Deliverables**:
- Plugin manifest schema: name, description, triggers, required tools, model compatibility
- Plugin registry with discovery and activation lifecycle
- Dynamic prompt injection: plugin activation injects prompt fragment as Mission sub-mission
- Model-specific format variants per plugin
- Hook system: pre-execution and post-execution hooks for tool calls
- Plugin sandbox: faulty plugins cannot crash the orchestrator
- Plugin development guide and example plugin

**Acceptance criteria**:
- Plugins can be added/removed without restarting the orchestrator
- Plugin prompt fragments respect the Mission composition rules (mereological coherence)
- A crashing plugin is isolated and logged, not propagated

**MxM mapping**: Mission (sub-mission composition) + Mind (context injection)

---

## Dependency Graph

```
P0: Tool Proxy ----+
                   |
P0: Permission ----+--> P1: Context Management ---> P2: Isolation
                   |
P0: Self-Exemption-+    P1: Output Parsing -------> P2: Streaming
                   |
                   +--> P1: Disposition Validation
                                                     P3: Skills (depends on P1)
```

- P0 components are independent of each other and can be built in parallel
- P1 components depend on P0 (tool calls must be intercepted before they can be context-managed or validated)
- P2 components depend on P1 (isolation wraps the full pipeline; streaming requires output parsing)
- P3 depends on P1 (plugins inject into the context management and validation pipeline)

## Milestones

| Milestone | Components | Gate |
|-----------|-----------|------|
| **M0: Safe to execute** | Tool Proxy + Permission Channel + Self-Exemption Prevention | All P0 acceptance criteria pass |
| **M1: Multi-model ready** | + Context Management + Output Parsing + Disposition Validation | Successful multi-turn sessions on 3+ model backends |
| **M2: Production hardened** | + Agent Isolation + Streaming | Isolation escape tests pass; streaming works across all backends |
| **M3: Extensible** | + Skill/Plugin System | Example plugin loads and operates correctly |
