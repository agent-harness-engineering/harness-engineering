# omaestro Implementation Roadmap

## Phase 0: Foundation (Skeleton)

**Goal:** Runnable Go binary with agent loop, single provider, basic tools.

**Deliverables:**
- Go module with CLI entry point
- Agent loop (OpenCode-derived): message -> provider -> tool dispatch -> loop
- Single provider integration (Anthropic Claude)
- Core tools: Read, Write, Edit, Bash, Glob, Grep
- SQLite session persistence
- PubSub event bus

**What this proves:** The core loop works. An agent can receive a prompt,
call tools, and produce results — same as OpenCode but in our codebase.

---

## Phase 1: Governance Layer

**Goal:** MxM gates integrated into tool dispatch pipeline.

**Deliverables:**
- Gate interface (Go) with fail-closed semantics
- Pre-commit gate (branch + content inspection)
- CDAE gate (destructive action detection)
- Force-push gate
- Append-only audit log (enforcement.log)
- Gate configuration per agent type

**What this proves:** Semantic governance works as compiled Go interfaces,
not shell hooks. Fail-closed behavior verified.

---

## Phase 2: Sandbox Integration

**Goal:** NemoClaw sandbox wrapping agent execution.

**Deliverables:**
- Sandbox Manager component
- Landlock filesystem policy per agent type
- seccomp profile per agent type
- Network namespace with per-agent egress policy
- Resource limits (cgroups) per agent type
- Container lifecycle: create -> run -> destroy

**What this proves:** Defense-in-depth works. Even with all gates bypassed,
the sandbox enforces structural constraints at the OS level.

---

## Phase 3: Multi-Provider + Agent Types

**Goal:** Multiple providers, agent type system, subagent spawning.

**Deliverables:**
- Provider interface (OpenCode-derived) with Anthropic, OpenAI, Google, local
- Agent type YAML declarations
- Type Resolver: type -> gates + sandbox + trace + tools
- Subagent spawning with per-agent model selection
- Cost tracking and rollup (parent <- child)
- Type composition (extends)

**What this proves:** The unifying abstraction works — one declaration,
four enforcement layers.

---

## Phase 4: Observability

**Goal:** Full tracing, evaluation, cost budgets.

**Deliverables:**
- Trace Collector subscribing to PubSub
- Hierarchical spans (agent -> tool calls -> subagent -> tool calls)
- Cost attribution per span
- Cost budget enforcement (warn at 80%, halt at 100%)
- OTLP export
- Basic evaluation framework (dataset-driven, custom evaluators)

**What this proves:** Everything is observable and measurable.

---

## Phase 5: UX + Polish

**Goal:** Production-grade user experience.

**Deliverables:**
- CLI with permission modes (Plan, AutoEdit, FullAuto)
- TUI (Bubble Tea) with operator approval, trace view, cost dashboard
- Project configuration (.omaestro/config.yaml)
- CLAUDE.md compatibility (load existing project instructions)
- Background agents with completion notification
- Multi-phase coordinator
- SendMessage to running agents

**What this proves:** omaestro is usable by humans, not just architecturally sound.

---

## Phase 6: Distribution

**Goal:** Easy deployment for teams and enterprises.

**Deliverables:**
- OCI blueprint distribution (NemoClaw-derived)
- Shared agent type libraries
- Policy preset marketplace
- Team configuration management
- Air-gap deployment mode (NIM, Ollama, vLLM)

---

## Dependencies Between Phases

```
Phase 0 (Foundation)
  |
  +-> Phase 1 (Governance) -- requires tool dispatch pipeline
  |     |
  |     +-> Phase 2 (Sandbox) -- requires governance to wrap
  |           |
  +-> Phase 3 (Multi-Provider + Types) -- requires loop + governance + sandbox
        |
        +-> Phase 4 (Observability) -- requires PubSub + types
              |
              +-> Phase 5 (UX) -- requires all layers working
                    |
                    +-> Phase 6 (Distribution) -- requires stable system
```

## Non-Goals (Explicitly Out of Scope)

- **IDE integration** — CLI/TUI first; IDE plugins are a separate project
- **Web UI** — terminal-native; web dashboard is a separate project
- **Model training** — omaestro orchestrates models, does not train them
- **Prompt engineering tools** — use existing tools (LangSmith, etc.)
- **RAG infrastructure** — use existing tools; omaestro can call them as tools
