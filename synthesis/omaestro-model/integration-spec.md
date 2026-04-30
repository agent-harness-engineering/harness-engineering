# omaestro Integration Specification

How each reference model maps to omaestro components.

## 4M -> Governance Engine

### What We Take
- **Morals module:** Prohibitions P1-P10, Obligations O1-O9, conflict resolution hierarchy
- **Mind module:** Confidence signaling, source protocol, circularity detection
- **Memory module:** Meta-context for cross-agent state, append-only audit log
- **Mission module:** Agent disposition framework, task scoping

### How It Integrates
4M gates become **Go interfaces** wrapping the tool dispatch pipeline:

```go
type Gate interface {
    // Check returns ALLOW or BLOCK with rationale
    Check(ctx context.Context, call ToolCall, session Session) Decision
    // Name returns the gate identifier for audit logging
    Name() string
}
```

Each agent type declares which gates apply. The Governance Engine runs all
applicable gates before tool execution. Any BLOCK stops execution.

**Key difference from current 4M:** Current implementation uses shell-script
PreToolUse hooks (fail-open — if hook crashes, tool executes). omaestro uses
compiled Go interfaces (fail-closed — if gate panics, tool does not execute).

### Audit Log Integration
Every gate decision writes to the append-only audit log:
```json
{
  "timestamp": "2026-04-30T14:30:00Z",
  "agent_id": "agent_20260430_1430",
  "agent_type": "coder",
  "gate": "cdae",
  "tool": "bash",
  "input": "git push --force origin main",
  "decision": "BLOCK",
  "rationale": "P4: force push to protected branch 'main'",
  "trace_id": "span_abc123"
}
```

---

## NemoClaw -> Sandbox Manager

### What We Take
- **Landlock LSM integration:** Filesystem access control per agent
- **seccomp profiles:** Syscall filtering per agent type
- **Network namespace management:** Per-agent egress control
- **Container lifecycle:** Create, configure, run, destroy
- **Policy-as-code:** YAML policy files with composable presets

### What We Do NOT Take
- NemoClaw's own agent loop (we use OpenCode-derived patterns)
- NemoClaw's inference routing (separate concern, may integrate later)
- NemoClaw's TUI (we build our own UX layer)
- NemoClaw's MCP server implementation (we use OpenCode's MCP patterns)

### How It Integrates
The Type Resolver converts agent type declarations to NemoClaw policy YAML:

```
agent-type: researcher
  sandbox:
    filesystem: read-only project root
    network: allow api.semanticscholar.org
    resources: { cpu: 1, memory: 2G, timeout: 30m }
```

Becomes:

```yaml
# Generated NemoClaw policy
sandbox:
  landlock:
    paths:
      - path: /project
        access: read
      - path: /tmp/workspace
        access: read-write
  seccomp:
    preset: restrictive
  network:
    default: deny
    allow:
      - endpoint: api.semanticscholar.org
        methods: [GET]
      - endpoint: api.anthropic.com  # provider
        methods: [POST]
  resources:
    cpu: 1
    memory: 2G
    timeout: 30m
```

### Container Lifecycle
1. **Spawn:** Sandbox Manager creates container from policy
2. **Mount:** Project workspace mounted per filesystem policy
3. **Run:** Agent loop executes inside container
4. **Monitor:** Resource usage tracked, limits enforced
5. **Destroy:** Container removed on completion, workspace unmounted

---

## OpenCode -> Agent Loop Engine

### What We Take
- **Agent loop architecture:** message -> provider -> tool dispatch -> persist -> loop
- **Multi-provider abstraction:** Provider interface supporting multiple LLM backends
- **Tool registry:** Typed tools with schemas and permission annotations
- **Session persistence:** SQLite-based conversation history with cost tracking
- **Auto-compact:** Context window management via summarization
- **Per-agent model selection:** Different agents can use different providers
- **Cost rollup:** Subagent costs aggregated to parent session
- **PubSub event bus:** Typed events for decoupled component communication

### What We Do NOT Take
- OpenCode's banned command list (replaced by NemoClaw sandbox — structural > blocklist)
- OpenCode's permission service (replaced by omaestro's multi-layer permission model)
- OpenCode's TUI (we build our own, informed by both OpenCode and NemoClaw TUI patterns)

### How It Integrates
The Agent Loop Engine is the core execution component. It:
1. Receives messages from user or parent agent
2. Calls provider via multi-provider abstraction
3. Dispatches tool calls through the governance + permission + sandbox pipeline
4. Persists results to SQLite
5. Publishes events to PubSub (consumed by Trace Collector)
6. Manages context window via auto-compact

**Key enhancement over OpenCode:** Tool calls pass through 4M gates and NemoClaw
sandbox before execution. OpenCode's pipeline is: permission -> execute. omaestro's
pipeline is: governance -> permission -> sandbox(execute) -> trace.

---

## Claude Code -> UX Layer

### What We Take (as patterns to reimplement)
- **Permission modes:** Plan (read-only), AutoEdit (edits OK), FullAuto (all OK)
- **Project configuration:** CLAUDE.md-style project instructions
- **Hook contracts:** PreToolUse, PostToolUse, Notification event model
- **Settings schema:** Allowlist/denylist patterns, project-level persistence
- **Agent spawning UX:** Background agents, multi-phase coordination, SendMessage

### What We Do NOT Take
- Any Claude Code source code (proprietary — behavioral description only)
- Claude-only model dependency (we use multi-provider via OpenCode patterns)
- Ephemeral session model (we use persistent sessions via OpenCode patterns)

### How It Integrates
The UX Layer provides the human interface to omaestro:
- CLI for interactive use (informed by Claude Code's CLI patterns)
- TUI for rich interaction (informed by OpenCode's Bubble Tea + NemoClaw's operator TUI)
- Permission prompts with context (what, why, which gate triggered)
- Project configuration loading (CLAUDE.md, .omaestro/config.yaml)

---

## LangSmith -> Trace Collector

### What We Take
- **Trace model:** Hierarchical spans with parent-child relationships
- **Run metadata:** Input, output, latency, token count, cost per span
- **Feedback model:** Scoring and annotation for quality tracking
- **Evaluation patterns:** Dataset-driven eval with custom evaluators
- **Export formats:** OTLP-compatible trace export

### What We Do NOT Take
- LangSmith SaaS platform (proprietary — we build local-first)
- LangSmith's prompt versioning (separate concern, may integrate later)
- LangSmith's annotation queues (enterprise feature, not core)

### How It Integrates
The Trace Collector subscribes to the PubSub event bus (OpenCode pattern):

```
PubSub events:
  AgentStart    -> open trace span
  ToolCallStart -> open child span (tool name, input)
  ToolCallEnd   -> close child span (output, latency, tokens)
  GateDecision  -> annotate span (gate name, decision, rationale)
  AgentEnd      -> close trace span (total cost, token summary)
```

Traces are stored locally (SQLite or JSON) and optionally exported to:
- OpenTelemetry collector (OTLP)
- File-based export (JSON lines)
- Custom evaluation pipelines

### Cost Budget Enforcement
The Trace Collector tracks cumulative cost per agent session. When a
configured budget threshold is reached:
- **Warn:** log + notification at 80% budget
- **Halt:** block further provider calls at 100% budget
- **Override:** user can manually approve continued execution
