# omaestro Architecture

## System Components

```
+------------------------------------------------------------------+
|                        omaestro Runtime                           |
|                                                                   |
|  +-------------------+    +-------------------+                   |
|  |  Agent Registry   |    |  Type Resolver    |                   |
|  |  (agent types,    |    |  (type -> gates   |                   |
|  |   declarations)   |--->|   + sandbox       |                   |
|  +-------------------+    |   + trace config  |                   |
|                           |   + tool set)     |                   |
|                           +--------+----------+                   |
|                                    |                              |
|  +-----------------------------+   |   +------------------------+ |
|  |      Governance Engine      |<--+-->|   Sandbox Manager      | |
|  |      (4M gates)             |       |   (NemoClaw policies)  | |
|  |                             |       |                        | |
|  |  - PreToolUse semantic gate |       |  - Container lifecycle | |
|  |  - Spawn approval gate     |       |  - Landlock policies   | |
|  |  - Content-aware blocking   |       |  - Network policy      | |
|  |  - Append-only audit log    |       |  - seccomp profiles    | |
|  +-----------------------------+       |  - Resource limits     | |
|                                        +------------------------+ |
|                                                                   |
|  +-----------------------------+   +---------------------------+  |
|  |      Agent Loop Engine      |   |   Trace Collector         |  |
|  |      (OpenCode patterns)    |   |   (LangSmith patterns)    |  |
|  |                             |   |                           |  |
|  |  - Multi-provider dispatch  |   |  - Hierarchical spans     |  |
|  |  - Tool registry + dispatch |   |  - Cost attribution       |  |
|  |  - Session persistence      |   |  - Token tracking         |  |
|  |  - Context management       |   |  - Eval dataset hooks     |  |
|  |  - Auto-compact             |   |  - Export (OTLP, JSON)    |  |
|  +-----------------------------+   +---------------------------+  |
|                                                                   |
|  +-----------------------------+                                  |
|  |      UX Layer               |                                  |
|  |      (Claude Code patterns) |                                  |
|  |                             |                                  |
|  |  - Permission modes         |                                  |
|  |  - CLAUDE.md / project cfg  |                                  |
|  |  - Hook contracts           |                                  |
|  |  - CLI + TUI interface      |                                  |
|  +-----------------------------+                                  |
+------------------------------------------------------------------+
```

## Request Flow: Tool Execution

```
User or parent agent sends message
  |
  v
Agent Loop Engine
  - Assembles system prompt (context paths + tool descriptions)
  - Calls provider with message history + tool schemas
  - Receives tool call from provider response
  |
  v
Governance Engine (4M)
  - Semantic gate inspects: tool name, arguments, content, branch, session state
  - Checks applicable prohibitions for this agent type
  - Decision: ALLOW / BLOCK (with rationale)
  - Decision logged to append-only audit
  |
  [BLOCKED] --> error returned to agent loop, agent adjusts
  [ALLOWED] --> continues
  |
  v
Permission Service (UX layer)
  - Checks allowlist/denylist patterns
  - If not pre-approved: prompt user (CLI/TUI)
  - Grant modes: single, persistent (session), persistent (project)
  |
  [DENIED] --> error returned to agent loop
  [APPROVED] --> continues
  |
  v
Sandbox Enforcement (NemoClaw)
  - Tool executes inside sandboxed container
  - Landlock: filesystem access per policy
  - seccomp: syscall filter per policy
  - Network: egress per whitelist
  - Resources: CPU/memory/time per budget
  |
  [SANDBOX VIOLATION] --> blocked at OS level, logged
  [EXECUTES] --> result captured
  |
  v
Trace Collector (LangSmith patterns)
  - Records: input, output, latency, tokens, cost
  - Links to parent span (if subagent)
  - Checks cost budget (warn/halt if exceeded)
  |
  v
Result returned to Agent Loop Engine
  - Persisted as tool_result message
  - History updated for next provider call
```

## Request Flow: Agent Spawn

```
Parent agent decides to spawn child
  |
  v
Governance Engine
  - Is this agent type allowed to spawn?
  - Is the child type appropriate for the task?
  - Audit: spawn request logged
  |
  [BLOCKED] --> spawn denied, parent informed
  [ALLOWED] --> continues
  |
  v
Type Resolver
  - Resolves child agent type declaration
  - Generates: tool allowlist, sandbox policy, trace config, provider config
  |
  v
Sandbox Manager
  - Creates container with resolved policy
  - Mounts workspace (read-only or read-write per type)
  - Configures network policy
  - Sets resource limits
  |
  v
Agent Loop Engine
  - Creates new agent instance inside container
  - Configures provider (may differ from parent)
  - Loads context paths
  - Runs agent loop until completion or timeout
  |
  v
On completion:
  - Result text returned to parent
  - Container destroyed
  - Cost rolled up to parent session
  - Trace span closed with final metrics
```

## Data Flow: Cross-Agent Context

```
Agent A writes to meta-context
  |
  v
Meta-Context Store (4M memory module)
  - File-based persistent state
  - Scoped: per-project, per-session, per-agent-type
  - Mounted read-only into other agent containers
  |
  v
Agent B reads meta-context at session start
  - Gets: what Agent A did, current state, blockers
  - No direct communication — async, file-based
```

## Component Provenance

| Component | Derived From | License | Implementation Approach |
|-----------|-------------|---------|----------------------|
| Agent Registry | Original | Ologos | YAML declarations |
| Type Resolver | Original | Ologos | Maps types to multi-layer configs |
| Governance Engine | 4M gates | Ologos | Go interfaces wrapping tool dispatch |
| Sandbox Manager | NemoClaw | Apache 2.0 | Container lifecycle + policy application |
| Agent Loop Engine | OpenCode patterns | MIT | Go agent loop with multi-provider |
| Trace Collector | LangSmith SDK patterns | MIT | PubSub subscriber emitting spans |
| UX Layer | Claude Code patterns | Public interfaces | Permission modes + project config |
| Meta-Context Store | 4M memory module | Ologos | File-based cross-agent state |

## Implementation Language

**Go** — chosen because:
1. OpenCode (primary agent loop reference) is Go — direct pattern reuse
2. NemoClaw integration via Go bindings to Landlock/seccomp
3. Single binary deployment (no runtime dependencies)
4. Strong concurrency primitives for multi-agent coordination
5. Type safety for gate interfaces (fail-closed by default)
