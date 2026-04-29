# Maestro Agent Architecture Specification

## 1. Overview

The Maestro Agent is a model-agnostic orchestration layer that implements the [4M cognitive architecture](../theory/4m-reference-architecture/index.md) (Mission, Mind, Morals, Memory) as a runtime harness for arbitrary LLM backends. It enforces the [Agentic Execution Control Plane](../docs/agentic-execution-control-plane.md) principle: **agent proposes, deterministic controls constrain, accountable humans authorize irreversible risk**.

This specification defines the components, interfaces, data flows, and governance integration required to achieve harness capabilities equivalent to Claude Code while remaining backend-agnostic.

## 2. Design Principles

1. **Model-agnostic by construction**: No component assumes a specific model's output format, token vocabulary, or API contract. Model-specific behavior is isolated in adapter modules.
2. **Fail-closed by default**: Any ambiguity, timeout, or parse failure results in denial, not execution.
3. **Executable enforcement**: Safety constraints are code-level gates (Morals), not prompt suggestions (Mind). Both exist; neither is sufficient alone.
4. **Single source of truth**: Each category of state has one authoritative store. Projections are derived and read-only.
5. **Observable**: Every component boundary is logged, traced, and testable in isolation.

## 3. System Architecture

### 3.1 Layered Architecture

```
+================================================================+
|                     External LLM Backends                       |
|  Claude API  |  OpenAI API  |  Gemini API  |  Local/Custom     |
+======+=========+=============+=============+====================+
       |         |             |             |
       +----+----+------+------+------+------+
            |           |             |
       +----v-----------v-------------v----+
       |       Streaming Adapter           |  Normalizes SSE across providers
       +-------------------+---------------+
                           |
       +-------------------v---------------+
       |       Output Validator            |  Post-generation output checks
       +-------------------+---------------+  (validate intent before parsing)
                           |
       +-------------------v---------------+
       |       Tool Output Parser          |  Extracts tool calls from model output
       +-------------------+---------------+
                           |
       +-------------------v---------------+
       |     Tool Interception Proxy       |  Normalizes, validates, authorizes,
       +-------------------+---------------+  logs, and routes tool calls
                           |
       +-------------------v---------------+
       |     Permission Gate               |  Allow / Deny / Escalate / Fail Closed
       +-------------------+---------------+
                           |
+======+===================v===============+====================+
|                  4M Governance Layer                           |
|                                                               |
|  Mission          Mind           Morals          Memory       |
|  (Objective       (Inference     (Policy         (State &     |
|   Scope)           Discipline)    Constraints)    Provenance) |
+======+===================+===============+====================+
                           |
       +-------------------v---------------+
       |          Gate Scripts              |
       |  CDAE | pre-commit | pre-push |   |
       |  pre-delete | pre-publish         |
       +-------------------+---------------+
                           |
       +-------------------v---------------+
       |         Execution Layer            |
       |  Read | Write | Mutate | Publish  |
       |  Delete | Exfiltrate              |
       +-------------------+---------------+
                           |
       +-------------------v---------------+
       |     Audit & Provenance Store      |
       |  Decision trace, tool calls,      |
       |  approvals, execution logs,       |
       |  rollback evidence                |
       +-----------------------------------+
```

### 3.2 Component Inventory

| Component | Layer | 4M Module | Priority |
|-----------|-------|-----------|----------|
| Streaming Adapter | Ingress | Means | P2 |
| Output Validator | Ingress | Morals + Mind | P1 |
| Tool Output Parser | Ingress | Means + Mind | P1 |
| Tool Interception Proxy | Control | Means + Morals | P0 |
| Permission Gate | Control | Morals | P0 |
| Self-Exemption Prevention | Control | Morals | P0 |
| Context Window Manager | State | Memory + Mind | P1 |
| Audit & Provenance Store | Observability | Morals + Memory | P1 |
| Agent Isolation | Boundary | Morals + Means | P2 |
| Skill/Plugin System | Extension | Mission + Mind | P3 |

## 4. Component Specifications

### 4.1 Tool Interception Proxy (P0)

**Purpose**: Sits between model tool calls and execution. Every tool call passes through this proxy; no tool call bypasses it.

**Interfaces**:

```
Input:  RawModelOutput  -- text or structured content from any LLM
Output: CanonicalToolCall | ValidationError

CanonicalToolCall {
    id:         string       -- unique call ID
    tool_name:  string       -- normalized tool name
    parameters: object       -- validated against tool schema
    source:     ModelFormat   -- which parser extracted this
    raw:        string       -- original model output (for audit)
    timestamp:  ISO8601
}
```

**Parsers**:

| Format | Detection | Extraction |
|--------|-----------|------------|
| OpenAI function calling | `tool_calls` array in response JSON | `function.name` + JSON.parse(`function.arguments`) |
| Anthropic tool_use | `content[].type == "tool_use"` | `name` + `input` object |
| Raw text action tags | Regex: `<tool_call>...</tool_call>` or configurable pattern | Tag content parsed as JSON or key-value pairs |

**Validation pipeline**:
1. Format detection (auto or configured per model)
2. Extraction via format-specific parser
3. Tool name lookup in registry (unknown tool = reject)
4. Parameter validation against tool's JSON schema
5. Authorization check against active policy context
6. Canonical representation emitted to Permission Gate
7. All calls logged to Audit & Provenance Store

**Failure mode**: Any parse or validation failure produces a `ValidationError` returned to the model as a tool error response, prompting self-correction. The tool is never executed.

### 4.2 Permission Gate (P0)

**Purpose**: Enforces human-in-the-loop approval for tool calls not covered by auto-allow policy. Implements the Morals module's deontic enforcement.

**Policy engine**:

```
PermissionPolicy {
    rules: [
        { pattern: "read_file",    action: "auto_allow" },
        { pattern: "write_file",   action: "escalate",   channels: ["cli", "web"] },
        { pattern: "shell_exec",   action: "escalate",   channels: ["cli", "web"] },
        { pattern: "rm_rf",        action: "deny" },
        { pattern: "git_push",     action: "escalate",   channels: ["cli", "web"] },
        ...
    ]
    default_action: "escalate"
    timeout_seconds: 300
    timeout_action: "deny"
}
```

**Decision semantics**:

| Decision | Meaning | Use Case |
|----------|---------|----------|
| `allow` | Auto-permitted by policy | Low-risk reads, approved tool patterns |
| `deny` | Permanently blocked by policy | Destructive operations, exfiltration attempts |
| `escalate` | Controlled human-in-the-loop | Write operations, mutations, publishes; includes policy exception handling for edge cases where the action may be valid but requires human judgment |
| `fail_closed` | Timeout or channel failure | No response within `timeout_seconds`; channel unreachable |

**Escalate semantics**: When a tool call is escalated, the relay channel presents the full tool call context (tool name, parameters, risk classification, active mission) to the human approver. The approver may:
- **Approve**: Tool call proceeds to execution
- **Deny**: Tool call is rejected with reason logged
- **Approve with modification**: Approver adjusts parameters before execution (e.g., narrowing a file glob)
- **Request policy exception**: For repeated escalations of the same pattern, the approver may create a session-scoped auto-allow rule (logged as a policy exception with expiry)

**Relay channels**:

| Channel | Use Case | Protocol |
|---------|----------|----------|
| CLI | Local development | stdin/stdout blocking prompt |
| Web UI | Remote/headless | WebSocket with session auth |
| Telegram/Slack | Mobile/async approval | Bot API with callback |
| Webhook | CI/CD integration | HTTP POST with HMAC signature |

**Fail-closed guarantee**: If no response arrives within `timeout_seconds`, the action is denied. The timeout is enforced by the orchestrator, not by the relay channel, preventing channel failures from causing hangs.

**Audit record**:

```
PermissionDecision {
    tool_call_id:  string
    decision:      "allow" | "deny" | "escalate" | "fail_closed"
    channel:       string
    approver:      string | null
    timestamp:     ISO8601
    context:       CanonicalToolCall
    risk_tier:     string        -- from execution risk classification
    exception:     object | null -- if policy exception was granted
}
```

### 4.3 Self-Exemption Prevention (P0)

**Purpose**: Prevents the agent process from modifying its own governance configuration. Implements the Morals module's defense-in-depth principle. Policy, config, and gate controls are read-only to the agent process.

**Enforcement layers**:

1. **Filesystem permissions**: Gate scripts, policy files, and governance configuration are owned by a non-agent user (e.g., `root` or `harness-admin`). Agent process runs as a restricted user with read-only access.

2. **Container boundary** (preferred): Governance configuration is bind-mounted read-only into the agent container:
   ```
   volumes:
     - ./governance:/etc/maestro-agent/governance:ro
     - ./gate-scripts:/etc/maestro-agent/gates:ro
   ```

3. **Process separation**: Configuration changes are applied by a separate management process with its own credentials. The agent process has no access to these credentials.

4. **Audit**: All access attempts to governance paths are logged to the Audit & Provenance Store. Unexpected access patterns trigger alerts.

**Invariant**: No code path exists from agent model output to governance configuration write.

### 4.4 Context Window Management (P1)

**Purpose**: Per-model token awareness with automatic compaction. Implements the Memory module's session continuity requirement.

**Tokenizer registry**:

| Model Family | Tokenizer | Library |
|-------------|-----------|---------|
| GPT-4/4o | cl100k_base | tiktoken |
| Claude | claude tokenizer | anthropic-tokenizer |
| Gemini | sentencepiece | sentencepiece |
| Llama/Mistral | sentencepiece (model-specific) | sentencepiece |
| Custom | configurable | pluggable |

**Three-zone budget** (per 4M Memory spec):

```
+--------------------------------------------------+
|  Pinned Zone (never evicted)                     |
|  - System prompt                                  |
|  - Tool definitions                               |
|  - Persistent memory block                        |
|  - Active Mission/Mind/Morals configuration       |
+--------------------------------------------------+
|  Summary Zone (regenerated on eviction)           |
|  - Running summary of evicted conversation turns  |
+--------------------------------------------------+
|  Verbatim Zone (oldest-first eviction)            |
|  - Recent messages in full fidelity               |
+--------------------------------------------------+
```

**Compaction trigger**: When total token count exceeds `compaction_threshold` (default: 80% of model's context limit):
1. Oldest verbatim messages are candidates for eviction
2. Evicted messages are summarized (LLM summarization with heuristic fallback)
3. Summary is prepended to the Summary Zone
4. Tool result content in retained messages older than N turns is replaced with truncation markers (observation masking)

**Configuration**:

```
ContextConfig {
    model:                 string    -- model identifier
    context_limit:         int       -- model's max tokens
    compaction_threshold:  float     -- default 0.80
    pinned_budget:         int       -- max tokens for pinned zone
    summary_budget:        int       -- max tokens for summary zone
    observation_mask_age:  int       -- turns before masking tool results
}
```

### 4.5 Structured Tool Output Parsing (P1)

**Purpose**: Reliable extraction of tool calls from any supported model's output format. Closely integrated with the Tool Interception Proxy (4.1) but focused on the parsing layer.

**Parser interface**:

```
Parser {
    can_parse(raw: string) -> bool        -- format detection
    parse(raw: string) -> CanonicalToolCall[] | ParseError
    format_result(result: ToolResult, model: ModelFormat) -> string
}
```

**Parser registry**: Model configuration maps to a parser chain. Auto-detection tries parsers in priority order; first match wins. Explicit configuration overrides auto-detection.

**Validation before execution**: Every parsed tool call is validated against the tool's JSON schema before being forwarded. This is the final safety gate before execution.

### 4.6 Output Validator (P1)

**Purpose**: Post-generation checks ensure model output meets quality, format, safety, and policy requirements defined by the active Mission. Positioned early in the flow (after streaming, before tool parsing) to validate intent before parsing mechanism.

**Validation dimensions**:

| Dimension | Check | Failure Action |
|-----------|-------|---------------|
| Schema compliance | Output conforms to declared JSON schema | Re-prompt with schema + error (retry) |
| Citation / provenance | Source attributions match required format; claims are grounded | Re-prompt with format spec (retry) |
| Policy compliance | No prohibited patterns; output aligns with active Morals constraints | Reject and log (escalate) |
| Tool-call legitimacy | Declared tool calls match expected patterns for active Mission | Re-prompt with correction (retry) |
| Destructive-action detection | Output proposes irreversible or high-risk operations | Flag for escalation (escalate immediately) |
| Instruction-conflict detection | Output contradicts system prompt or active Mission directives | Re-prompt with conflict highlighted (retry) |
| Rollback requirement detection | Output modifies state that requires rollback capability | Verify rollback path exists before proceeding (escalate if absent) |
| Confidence tags | Required confidence markers present | Re-prompt with instruction (retry) |
| Length bounds | Response within min/max token range | Re-prompt with bounds (retry) |

**Severity-gated retry logic**:
1. Generate response from model
2. Run validation rules for active Mission profile
3. On failure, classify severity:
   - **Format errors** (schema, citations, confidence tags, length): re-prompt with specific error (include the failed output and the rule it violated). Max 3 retries per generation.
   - **Suspicious actions** (destructive-action detection, policy violations, instruction conflicts): escalate immediately to human via Permission Gate. No automatic retry.
   - **Missing rollback path**: escalate to human with rollback requirement details.
4. After max retries on format errors: escalate to human via Permission Gate rather than emitting invalid output.

**Per-mission profiles**: Different Missions activate different validation profiles. A coding Mission may require JSON schema compliance; a research Mission may require citation format; a creative Mission may relax format constraints.

## 5. Execution Risk Classification

Every tool call is classified into a risk tier based on its capability class. This classification drives Permission Gate policy, audit verbosity, and escalation behavior.

| Tier | Capability Class | Description | Examples | Default Gate Action |
|------|-----------------|-------------|----------|-------------------|
| **T0** | Read | Observe state without modification | `read_file`, `git_status`, `grep`, `ls` | `allow` |
| **T1** | Write | Create or overwrite files in sandbox | `write_file`, `create_dir`, `save_config` | `escalate` |
| **T2** | Mutate | Modify existing state | `edit_file`, `git_commit`, `rename`, `chmod` | `escalate` |
| **T3** | Publish | Expose content beyond the sandbox boundary | `git_push`, `send_email`, `post_to_api`, `deploy` | `escalate` |
| **T4** | Delete | Destroy data or state | `rm`, `git_reset --hard`, `drop_table`, `truncate` | `escalate` (deny by default for recursive/bulk) |
| **T5** | Exfiltrate | Transfer data to external systems | `curl_upload`, `scp`, `webhook_with_payload` | `deny` (requires explicit policy exception) |

**Risk tier properties**:
- Each tier inherits the audit requirements of all lower tiers (T3 logs everything T0-T2 would log, plus publish-specific fields)
- Higher tiers require more context in the escalation prompt (T4/T5 include rollback plan and blast radius estimate)
- Policy rules may reference tiers directly: `{ tier: "T3+", action: "escalate" }`
- Mission profiles may restrict available tiers (e.g., a research Mission may cap at T1)

### 5.1 Audit & Provenance Store (P1)

**Purpose**: First-class component providing immutable, append-only logging of all governance-relevant events across the Maestro Agent pipeline. Enables post-hoc review, compliance evidence, and rollback decisions.

**Logged events**:

| Event Type | Source Component | Payload |
|------------|-----------------|---------|
| `tool_call_parsed` | Tool Output Parser | Canonical tool call, source format, raw model output |
| `tool_call_validated` | Tool Interception Proxy | Validation result, schema errors if any |
| `permission_decision` | Permission Gate | Decision, channel, approver, risk tier, exception details |
| `output_validation` | Output Validator | Rule results, severity, retry count |
| `gate_script_result` | Gate Scripts | Script name, pass/fail, execution time |
| `execution_result` | Execution Layer | Tool name, parameters, result, duration, exit code |
| `escalation` | Permission Gate / Output Validator | Escalation reason, context, human response |
| `policy_exception` | Permission Gate | Exception scope, expiry, approver, justification |
| `governance_access` | Self-Exemption Prevention | Access attempt details, allowed/denied |

**Storage requirements**:
- Append-only: events are never modified or deleted during a session
- Tamper-evident: each entry includes a hash chain linking to the previous entry
- Queryable: supports filtering by event type, component, time range, and tool name
- Exportable: full session audit trail can be exported as JSON or structured log format

**Interface**:

```
AuditStore {
    log(event: AuditEvent) -> void
    query(filter: AuditFilter) -> AuditEvent[]
    export(session_id: string, format: "json" | "jsonl") -> string
    verify_integrity(session_id: string) -> bool
}
```

## 6. Data Flow

### 6.1 Standard Tool Call Flow

```
1.  User sends message
2.  Context Manager assembles prompt (pinned + summary + verbatim + user message)
3.  Prompt sent to LLM backend via Streaming Adapter
4.  Streaming Adapter normalizes response into internal event stream
5.  Output Validator checks model response for policy compliance,
    destructive-action signals, and instruction conflicts
    (validate intent before parsing mechanism)
6.  Tool Output Parser extracts tool calls from validated stream
7.  Tool Interception Proxy normalizes, validates, authorizes, and logs each tool call
8.  Permission Gate evaluates policy with risk tier:
    a. allow -> proceed to execution
    b. escalate -> relay to human with full context, await decision
    c. deny -> return error to model
    d. fail_closed -> deny on timeout (fail-closed)
9.  4M Governance Layer applies mission, mind, morals, and memory constraints
10. Gate Scripts run pre-execution checks (CDAE, pre-commit, etc.)
11. Execution Layer performs the action
12. Result logged to Audit & Provenance Store
13. Result returned to model in expected format
14. Memory persists conversation state
15. Response delivered to user
```

### 6.2 4M Cross-Cutting Channels

The four cross-cutting channels from the 4M spec operate within this flow:

| Channel | Trigger | Effect in Maestro Agent |
|---------|---------|----------------------|
| Memory to Mind | Mid-reasoning recall | Model invokes memory search tool; Context Manager injects results |
| Mind to Memory | Persistence decision | Model emits save action tag; Memory module writes to persistent store |
| Mission to Morals | Sub-mission activation | Active sub-mission selects permission policy profile and validation rules |
| Morals to Mission | Repeated constraint violations | Circuit breaker triggers sub-mission reconfiguration or tool subset adjustment |

## 7. Governance Integration

### 7.1 4M Module Mapping

Every Maestro Agent component maps to a 4M module, ensuring no governance gap:

| 4M Module | Maestro Agent Components |
|-----------|------------------------|
| **Mission** | Skill/Plugin System, sub-mission router, intent classifier |
| **Mind** | Context Window Manager (reasoning quality), cognitive prompt fragments, inference mode selection |
| **Morals** | Permission Gate, Self-Exemption Prevention, Output Validator, Agent Isolation, Gate Scripts |
| **Memory** | Context Window Manager (session continuity), persistent store, observation masking, running summaries, Audit & Provenance Store |
| **Means** | Tool Interception Proxy, Streaming Adapter, Execution Layer, Tool Output Parser |

### 7.2 AECP Integration

The Maestro Agent implements the Agentic Execution Control Plane architecture:

| AECP Component | Maestro Agent Mapping |
|---------------|---------------------|
| Agent | LLM backend + Mind configuration |
| QA Agent | Output Validator (automated QA) |
| Policy Broker | Permission Gate + Gate Scripts |
| Execution Broker | Tool Interception Proxy + Execution Layer |

### 7.3 Gate Scripts

Gate scripts are deterministic pre-execution checks that operate between the Permission Gate and the Execution Layer:

| Gate | Trigger | Check |
|------|---------|-------|
| CDAE (Contextual Disposition Audit Engine) | Every tool call | Disposition alignment with active Mission |
| pre-commit | Git commit operations | Code quality, secrets detection, policy compliance |
| pre-push | Git push operations | Branch protection, remote policy |
| pre-delete | Destructive file/data operations | Backup verification, confirmation |
| pre-publish | External publication actions | Content review, sensitivity check |

## 8. Configuration

### 8.1 Maestro Agent Configuration File

```yaml
# maestro-agent.yaml
version: "1.0"

backend:
  provider: "anthropic"           # or "openai", "google", "custom"
  model: "claude-sonnet-4-6"
  api_endpoint: "https://api.anthropic.com/v1"
  streaming: true

governance:
  mission:
    base: "missions/base.md"
    sub_missions:
      - "missions/coding.yaml"
      - "missions/research.yaml"
  morals:
    permission_policy: "policies/permissions.yaml"
    gate_scripts: "gates/"
    self_exemption_prevention: true
    risk_classification: "policies/risk-tiers.yaml"
  memory:
    persistent_store: "sqlite:///memory.db"
    session_store: "memory"
    compaction_threshold: 0.80
  mind:
    cognitive_guides: "guides/"
    inference_modes: ["deductive", "inductive", "abductive"]

audit:
  store: "sqlite:///audit.db"
  export_format: "jsonl"
  integrity_check: true

isolation:
  enabled: true
  sandbox_dirs: ["/workspace"]
  resource_limits:
    memory: "2g"
    cpus: 2.0
    timeout_per_call: 120

plugins:
  registry: "plugins/"
  auto_load: true
```

## 9. Testing Strategy

Each component boundary and cross-cutting channel is an independently testable interface:

| Test Category | Scope | Method |
|--------------|-------|--------|
| Parser correctness | Tool Output Parser | Unit tests: known model outputs -> expected canonical calls |
| Permission enforcement | Permission Gate | Integration tests: tool call -> policy evaluation -> expected decision |
| Escalation flow | Permission Gate | Integration tests: escalate -> human response -> execution or denial |
| Risk classification | Execution Risk | Unit tests: tool call -> expected tier assignment |
| Self-exemption | Isolation boundary | Escape tests: agent attempts config write -> blocked and logged |
| Context management | Context Window Manager | Property tests: no overflow, pinned content preserved, summaries coherent |
| Output validation | Output Validator | Unit tests: known outputs -> validation pass/fail per rule set |
| Severity gating | Output Validator | Unit tests: format errors retry, suspicious actions escalate |
| Audit integrity | Audit & Provenance Store | Integration tests: hash chain verification, append-only enforcement |
| End-to-end safety | Full pipeline | Adversarial tests: prompt injection, tool abuse, escape attempts |
| Cross-cutting channels | 4M channels | Simulation tests: trigger conditions -> expected payload and downstream effect |
