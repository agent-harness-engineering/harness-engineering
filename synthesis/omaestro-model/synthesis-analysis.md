# omaestro Synthesis Analysis

Cross-cutting analysis of four reference models (Claude Code, OpenCode, NemoClaw, LangSmith)
and one original framework (MxM) that justifies every architectural decision in omaestro.

This document is the **analytical foundation** underneath the architecture docs. Those docs
describe what omaestro is. This doc argues *why*, with evidence from the landscape.

---

## 1. Convergence Matrix

Where two or more models independently solve the same problem the same way.
Convergent evolution = validated pattern, low-risk design choice.

### 1.1 Tool Dispatch as Universal Interception Point

**Models:** Claude Code, OpenCode, NemoClaw, MxM

All four models that involve agent execution treat tool dispatch as the primary
control surface. The agent decides to call a tool; governance happens in the gap
between decision and execution.

| Model | Interception Mechanism | When |
|-------|----------------------|------|
| Claude Code | PreToolUse/PostToolUse hooks | Before/after every tool call |
| OpenCode | Permission service check | Before every modifying tool call |
| NemoClaw | Sandbox policy enforcement | At syscall level during execution |
| MxM | Semantic gate scripts | Before tool execution via hooks |

**omaestro conclusion:** Tool dispatch is the universal governance surface. Every
safety, governance, and observability concern attaches here. The pipeline is:
governance gate -> permission check -> sandbox(execute) -> trace.

**Confidence: VERY HIGH** — no model disputes this pattern.

---

### 1.2 Read-Only Subagents as Privilege Tier

**Models:** Claude Code (explore/plan agents), OpenCode (task agent)

Both agent platforms independently invented a restricted subagent type that can
only read, not modify. Same rationale: exploratory queries don't need write access,
and restricting them reduces blast radius.

| Model | Read-Only Agent | Tools |
|-------|----------------|-------|
| Claude Code | explore, plan | Glob, Grep, Read (no Edit, Write, Bash) |
| OpenCode | task | Glob, Grep, Ls, View, Sourcegraph (no Bash, Edit, Write) |

**omaestro conclusion:** The researcher agent type is directly validated by this
convergence. Two independent projects arrived at the same privilege split.

**Confidence: HIGH** — convergent evolution across projects with no shared codebase.

---

### 1.3 Process-Level Isolation Is Insufficient

**Models:** Claude Code (acknowledged gap), OpenCode (acknowledged gap), NemoClaw (solved)

Both agent platforms document that their agents share the parent's filesystem and
network access. Both explicitly acknowledge this as a safety gap. NemoClaw exists
specifically to solve this gap via container isolation.

| Model | Isolation Level | Acknowledged Gap? |
|-------|----------------|------------------|
| Claude Code | Process-level only | Yes — "primary gap that NemoClaw fills" |
| OpenCode | Process-level only | Yes — 6 structural gaps documented |
| NemoClaw | Container + Landlock + seccomp + netns | N/A — provides the solution |

**omaestro conclusion:** OS-level sandboxing (NemoClaw's approach) is not optional.
Both agent platforms agree their isolation is insufficient. This is the strongest
possible validation for the Sandbox Manager component.

**Confidence: VERY HIGH** — the gap is acknowledged by the systems that have it.

---

### 1.4 User Approval as Gate, Not Policy

**Models:** Claude Code (permission modes), OpenCode (permission service)

Both platforms treat user approval as a runtime gate — the user says yes or no to
a specific action. Neither treats it as a policy that can reason about context.
Both acknowledge this limitation.

| Model | Approval Mechanism | Context-Aware? |
|-------|-------------------|---------------|
| Claude Code | CLI y/n prompt, allowlist patterns | No — pattern match only |
| OpenCode | TUI dialog, grant/deny/persist | No — tool+action+path only |

**omaestro conclusion:** Permission is a necessary but insufficient layer. It catches
what the user doesn't want, but it doesn't understand *why* something is dangerous.
That's the Governance Engine's job (MxM semantic gates). Permission Service and
Governance Engine are complementary, not redundant.

**Confidence: HIGH** — both platforms' permission models are content-blind.

---

### 1.5 Append-Only Audit as Distinct Concern

**Models:** MxM (enforcement.log), LangSmith (trace store)

Both independently implement append-only records of decisions, though at different
granularity levels. MxM logs gate decisions with rationale. LangSmith logs execution
spans with metrics.

| Model | What's Logged | Format | Queryable? |
|-------|-------------|--------|-----------|
| MxM | Gate decisions + rationale | Flat file, append-only | grep only |
| LangSmith | Execution spans + metrics + feedback | Structured store | Full query + filter |

**Contrast:** Claude Code and OpenCode have *no* persistent audit trail. Tool
approvals are ephemeral (session-only).

**omaestro conclusion:** Audit is not a nice-to-have; it's the foundation for
compliance, incident review, and governance improvement. Combine MxM's decision
rationale with LangSmith's structured tracing to get both *what happened* and *why
it was allowed*.

**Confidence: HIGH** — both audit-capable models agree on append-only semantics.

---

### 1.6 Context Injection at Session Start

**Models:** Claude Code (CLAUDE.md), OpenCode (context paths)

Both agent platforms load project-level instructions at session start from
conventional file paths. The pattern is identical: read files, inject into system
prompt, agent operates under those constraints.

| Model | Files Loaded | Mechanism |
|-------|-------------|-----------|
| Claude Code | CLAUDE.md, CLAUDE.local.md | Auto-loaded into system prompt |
| OpenCode | CLAUDE.md, opencode.md, .cursorrules, copilot-instructions.md | Concatenated as system context |

**omaestro conclusion:** Project configuration via conventional files is a validated
pattern. omaestro uses the same approach (.omaestro/config.yaml + CLAUDE.md).

**Confidence: HIGH** — universal pattern, no disagreement.

---

## 2. Divergence Register

Where models disagree or take fundamentally different approaches.
Each divergence is a **design decision** omaestro must make, with evidence for each side.

### 2.1 Extensibility: Shell Hooks vs Go Interfaces

| | Claude Code | OpenCode |
|---|-----------|----------|
| **Mechanism** | Shell-script hooks (PreToolUse, PostToolUse) | Go `BaseTool` interface |
| **Execution** | Subprocess with JSON on stdin | In-process method call |
| **Fail mode** | **Fail-open** — hook crash = tool executes | **Fail-closed possible** — panic = tool blocked |
| **Type safety** | None — string matching, exit codes | Full — compiled interfaces, typed events |
| **Extensibility** | Any language, any script | Go only |
| **MxM integration** | Shell scripts (current implementation) | Go wrapper functions |

**omaestro decision: Go interfaces (fail-closed).**

**Justification:**
- The MxM safety analysis identifies fail-open as a critical weakness. A crashing
  hook silently allowing a dangerous operation is worse than a crashing gate
  blocking a safe operation.
- OpenCode's Go interfaces enable compiled, type-safe gate checks. The gate contract
  is enforced at compile time, not runtime string matching.
- Shell hooks are still supported as a UX compatibility layer (Claude Code users
  expect them), but they wrap the Go interface, not the other way around.

**Trade-off acknowledged:** Shell hooks are more accessible (any language). Go
interfaces require recompilation. We accept this trade-off because safety gates
are infrastructure, not user scripts.

---

### 2.2 Session Persistence: Ephemeral vs Durable

| | Claude Code | OpenCode |
|---|-----------|----------|
| **Storage** | Process memory only | SQLite |
| **Survives restart?** | No (unless --resume) | Yes |
| **Cross-session query** | Not possible | SQL queries over all sessions |
| **Cost tracking** | Not tracked | Per-session token count + cost |
| **Audit surface** | None | Messages persist (but not decisions) |

**omaestro decision: SQLite (OpenCode pattern).**

**Justification:**
- Audit (convergence 1.5) requires persistent records. Ephemeral sessions cannot
  be audited after the fact.
- Cost tracking requires persistent state. You can't enforce budgets if you can't
  count cumulative cost.
- Session persistence turns an agent from a stateless tool into an auditable actor.
  This is a prerequisite for enterprise governance.
- Claude Code's `--resume` is an acknowledgment that persistence is useful, but
  it's opt-in and limited. omaestro makes it default.

**Trade-off acknowledged:** SQLite adds disk I/O and storage requirements. For
single-session CLI use, this is overhead. We accept this because the governance
use case requires it.

---

### 2.3 Network Restriction: Blocklist vs Sandbox

| | OpenCode | NemoClaw |
|---|---------|----------|
| **Mechanism** | Banned command list (curl, wget, nc...) | Default-deny network namespace |
| **Enforcement level** | Application (string matching) | OS kernel (syscall/netns) |
| **Bypassable?** | Yes — `python -c "import urllib..."` | No — requires kernel exploit |
| **Coverage** | Common CLI tools only | All network access from all processes |
| **Operator burden** | Zero (built-in list) | Moderate (whitelist management) |

**omaestro decision: Both — blocklist as Layer 4, sandbox as Layer 5.**

**Justification:**
- The blocklist catches 90% of cases cheaply (zero config, zero latency).
- The sandbox catches the remaining 10% that bypass the blocklist.
- Defense in depth means using both, not choosing one.
- OpenCode's analysis explicitly acknowledges the blocklist is bypassable.
  Including it as a layer is honest about its limitations while capturing its value.

**Trade-off acknowledged:** Two network restriction layers adds complexity. But the
alternative (sandbox-only) loses the cheap early-catch that reduces sandbox noise.

---

### 2.4 Agent Communication: One-Shot vs Interactive

| | Claude Code | OpenCode |
|---|-----------|----------|
| **Subagent communication** | SendMessage for follow-ups | One-shot: prompt in, result out |
| **Background agents** | Supported (detached) | Not supported |
| **Multi-phase coordination** | Phase-based dependency ordering | Not supported |
| **Agent hierarchy** | Arbitrary depth | Two-level (Coder -> Task) |

**omaestro decision: Both — one-shot default, interactive available.**

**Justification:**
- One-shot is simpler, easier to sandbox (container lifecycle = agent lifecycle),
  easier to trace (one span = one agent invocation).
- SendMessage is necessary for foreground interactive agents where the parent
  needs to refine the task based on intermediate results.
- Background agents are necessary for long-running tasks (build, test, deploy).
- Multi-phase coordination is necessary for complex workflows.
- OpenCode's simplicity is the default; Claude Code's richness is available
  when needed.

---

### 2.5 Permission Granularity: Pattern-Based vs All-or-Nothing

| | Claude Code | OpenCode |
|---|-----------|----------|
| **Pre-configuration** | Allowlist/denylist patterns in settings.json | None |
| **Modes** | Plan / AutoEdit / FullAuto | Ask-all / Auto-approve-all |
| **Persistence** | Per-project in config file | Per-session only |
| **Granularity** | Tool + input pattern | Tool + action + path |

**omaestro decision: Pattern-based (Claude Code), with OpenCode's TUI.**

**Justification:**
- Graduated autonomy matters. "Edits OK but bash asks" (AutoEdit) is the sweet
  spot for most development work. All-or-nothing forces users to either accept
  constant interruptions or give up all control.
- Pre-configured rules reduce approval fatigue for known-safe operations
  (e.g., always allow `npm test`, always deny `rm -rf`).
- Per-project persistence means each project gets the right trust level
  without reconfiguration each session.
- OpenCode's TUI is better than Claude Code's CLI prompt for presenting approval
  decisions with context. We take the permission model from Claude Code and the
  presentation from OpenCode.

---

### 2.6 Model Flexibility: Single Provider vs Multi-Provider

| | Claude Code | OpenCode | NemoClaw |
|---|-----------|----------|----------|
| **Providers** | Anthropic only | 11+ providers | NVIDIA Cloud, NIM, vLLM, Ollama |
| **Per-agent model** | Inherited from parent | Configurable per agent type | Configurable |
| **Air-gap support** | No | Yes (local endpoints) | Yes (NIM local) |

**omaestro decision: Multi-provider (OpenCode abstraction + NemoClaw routing).**

**Justification:**
- Model-agnostic governance is a core omaestro principle. Governance rules should
  not depend on which model is executing them.
- Air-gapped operation is a hard requirement for some deployment contexts.
- Per-agent model selection enables cost optimization (cheap model for summarization,
  expensive model for coding).
- OpenCode's Provider interface is Go-native and already supports the widest range.
  NemoClaw's inference routing adds sandbox-integrated model access.

---

## 3. Cross-Gap Matrix

Unified view of which capabilities each model provides and which it lacks.
Read column-wise to see what a model needs; read row-wise to see who provides it.

| Capability | Claude Code | OpenCode | NemoClaw | LangSmith | MxM |
|-----------|:-----------:|:--------:|:--------:|:---------:|:---:|
| **Agent loop** | Has | Has | Has | — | — |
| **Tool dispatch** | Has | Has | — | — | Gates |
| **Multi-provider** | — | Has | Has | — | — |
| **Filesystem sandbox** | — | — | Has | — | — |
| **Network sandbox** | — | Partial* | Has | — | — |
| **Resource limits** | — | — | Has | — | — |
| **Semantic gates** | — | — | — | — | Has |
| **Content-aware blocking** | — | — | — | — | Has |
| **Permission modes** | Has | Partial** | — | — | — |
| **Hook extensibility** | Has | — | — | — | — |
| **Hierarchical tracing** | — | — | — | Has | — |
| **Eval pipelines** | — | — | — | Has | — |
| **Cost attribution** | — | Has | — | Has | — |
| **Append-only audit** | — | — | — | — | Has |
| **Human feedback loop** | — | — | — | Has | — |
| **Session persistence** | — | Has | — | — | — |
| **LSP integration** | — | Has | — | — | — |
| **PubSub event bus** | — | Has | — | — | — |
| **Cross-agent context** | — | — | — | — | Has |
| **Blueprint distribution** | — | — | Has | — | — |
| **Background agents** | Has | — | — | — | — |
| **Multi-phase coordination** | Has | — | — | — | — |

\* OpenCode has a banned command blocklist, not a sandbox.
\** OpenCode has ask-all or auto-approve-all, no intermediate modes.

### Cross-Fill Summary

Each model fills gaps in the others. The synthesis value is in the *combination*:

| Gap Filler | Fills Gaps In | Key Gaps Filled |
|-----------|--------------|-----------------|
| **NemoClaw** | Claude Code, OpenCode | Filesystem sandbox, network sandbox, resource limits |
| **MxM** | Claude Code, OpenCode, LangSmith | Semantic gates, content-aware blocking, audit, governance |
| **OpenCode** | Claude Code, LangSmith | Multi-provider, session persistence, PubSub, LSP, cost tracking |
| **Claude Code** | OpenCode, LangSmith | Permission modes, hook extensibility, background agents, multi-phase |
| **LangSmith** | All others | Tracing, eval pipelines, human feedback, cost attribution |

**Key insight:** No model is redundant. Each provides at least one capability that
no other model provides. Removing any model from the synthesis leaves a gap that
cannot be filled by the remaining models.

---

## 4. Design Justification Trace

For each major omaestro architectural decision, which model(s) informed it,
what alternatives were considered, and why this choice was made.

### 4.1 Go as Implementation Language

| Factor | Evidence |
|--------|---------|
| **Primary influence** | OpenCode is Go — direct pattern reuse for agent loop |
| **Supporting evidence** | NemoClaw integration via Go bindings to Landlock/seccomp |
| **Alternative considered** | TypeScript (Claude Code's ecosystem, NemoClaw plugin) |
| **Why Go wins** | Single binary, type-safe gate interfaces, concurrency primitives, syscall access |
| **Why not TypeScript** | No direct Landlock/seccomp bindings, runtime dependency, fail-open semantics |

---

### 4.2 Fail-Closed Gate Semantics

| Factor | Evidence |
|--------|---------|
| **Primary influence** | MxM safety analysis — fail-open hooks identified as critical weakness |
| **Supporting evidence** | OpenCode's Go interfaces demonstrate fail-closed is implementable |
| **Negative evidence** | Claude Code's shell hooks crash = tool executes (documented behavior) |
| **Alternative considered** | Shell hooks with watchdog (restart crashed hooks) |
| **Why fail-closed wins** | A blocked safe operation is recoverable; an allowed dangerous operation may not be |

---

### 4.3 Agent-Type-to-Four-Layer Declaration

| Factor | Evidence |
|--------|---------|
| **Primary influence** | Original omaestro design (no model has this) |
| **Informed by** | Claude Code's agent types (tool allowlists), OpenCode's Coder/Task split, NemoClaw's policy presets |
| **Why novel** | No model connects agent type to sandbox policy, governance config, and trace config simultaneously |
| **Alternative considered** | Separate configuration for each layer |
| **Why single declaration** | Reduces configuration drift between layers; one source of truth for agent constraints |

---

### 4.4 PubSub as Internal Event Bus

| Factor | Evidence |
|--------|---------|
| **Primary influence** | OpenCode's `pubsub.Broker[T]` — typed events for component decoupling |
| **Supporting evidence** | LangSmith tracing requires an event source; PubSub is the natural integration |
| **Contrast** | Claude Code uses shell hooks (subprocess-based, not event-based) |
| **Alternative considered** | Hook-based extensibility (Claude Code pattern) |
| **Why PubSub wins** | Typed events, in-process, supports multiple subscribers, no subprocess overhead |

---

### 4.5 SQLite for Session + Audit Storage

| Factor | Evidence |
|--------|---------|
| **Primary influence** | OpenCode's SQLite session persistence |
| **Supporting evidence** | MxM audit needs persistent storage; LangSmith traces need queryable storage |
| **Contrast** | Claude Code sessions are ephemeral |
| **Why SQLite** | Single-file database, no server, SQL queries for analysis, ACID guarantees |
| **Why not flat files** | MxM's enforcement.log is grep-only; structured queries are essential for analysis |

---

### 4.6 Six-Layer Defense Stack

| Factor | Evidence |
|--------|---------|
| **Layer 1 (Model Safety)** | All models assume this exists; none controls it |
| **Layer 2 (Semantic Gates)** | MxM — original, no other model has it |
| **Layer 3 (Permission Service)** | Claude Code + OpenCode convergence (section 1.4) |
| **Layer 4 (Command Filtering)** | OpenCode — cheap early-catch, explicitly acknowledged as bypassable |
| **Layer 5 (OS Sandbox)** | NemoClaw — structural enforcement, validated by convergence (section 1.3) |
| **Layer 6 (Audit Trail)** | MxM + LangSmith convergence (section 1.5) |
| **Why six, not fewer** | Each layer catches threats the others miss (see defense-in-depth.md threat matrix) |
| **Why not more** | Every additional layer adds latency; six covers the threat model without excess |

---

### 4.7 Unified TUI (Bubble Tea)

| Factor | Evidence |
|--------|---------|
| **Primary influence** | OpenCode's Bubble Tea TUI (Go-native, composable) |
| **Supporting evidence** | NemoClaw's operator TUI (approval workflow for sandbox violations) |
| **Contrast** | Claude Code's CLI prompt (minimal context) |
| **Why unified** | Permission requests, gate decisions, sandbox violations, and traces should be visible in one interface |
| **Why Bubble Tea** | Go-native (matches implementation language), composable views, mouse support |

---

## 5. Novel Contribution Register

What omaestro provides that **no** source model has.
This is the IP story — what Ologos Corp invented vs synthesized.

### 5.1 Agent-Type-to-Multi-Layer Resolution (NOVEL)

**No source model** maps a single agent type declaration to governance config +
sandbox policy + trace config + loop config simultaneously. Each model handles
at most one of these layers.

- Claude Code: agent type -> tool allowlist (one layer)
- OpenCode: agent type -> tool set + model selection (two layers)
- NemoClaw: policy preset -> sandbox config (one layer)
- LangSmith: project -> trace config (one layer)

omaestro's Type Resolver takes a single YAML declaration and generates all four.
This is the core architectural innovation.

**IP classification: Original Ologos Corp design.**

---

### 5.2 Compiled Governance Gates with Fail-Closed Semantics (NOVEL)

MxM has semantic gates (shell-script hooks, fail-open).
OpenCode has Go tool interfaces (compiled, but no governance logic).

omaestro combines MxM's *semantic analysis* with OpenCode's *compiled interface pattern*
to produce governance gates that are both context-aware AND fail-closed. No source
model has this combination.

**IP classification: Original synthesis — MxM semantics + OpenCode implementation pattern.**

---

### 5.3 Cost Budget Enforcement via Trace-Sandbox Integration (NOVEL)

LangSmith tracks cost per trace. NemoClaw enforces resource limits via cgroups.
Neither connects inference cost to resource governance.

omaestro's Trace Collector monitors cumulative inference cost per agent and can
halt execution when a budget threshold is reached. This bridges LangSmith's
cost attribution with NemoClaw's enforcement model.

**IP classification: Original synthesis — LangSmith cost model + NemoClaw enforcement pattern.**

---

### 5.4 Governance-Aware Observability (NOVEL)

LangSmith traces execution but has no concept of governance. MxM audits governance
but has no structured tracing. No model combines them.

omaestro annotates every trace span with governance metadata: which gates fired,
what they decided, why. This makes governance decisions first-class objects in
the observability pipeline, queryable and evaluable.

**IP classification: Original synthesis — LangSmith tracing + MxM audit.**

---

### 5.5 Evaluation-Driven Governance Improvement (NOVEL)

LangSmith has evaluation pipelines (dataset-driven, custom evaluators, regression testing).
MxM has governance rules (static, manually maintained).

omaestro applies LangSmith's evaluation patterns *to governance rules themselves*:
gate decisions become evaluable targets, known-good/known-bad tool calls become
test datasets, governance changes are regression-tested before deployment.

No source model evaluates its own governance.

**IP classification: Original synthesis — LangSmith eval pattern + MxM governance.**

---

### 5.6 Cross-Agent Context via Sandbox-Isolated Mounts (NOVEL)

MxM has meta-context files for cross-agent state. NemoClaw has per-container
filesystem isolation. No model combines them.

omaestro mounts meta-context files as read-only volumes in agent containers.
Agent A writes context; Agent B's sandbox mounts it read-only. The sandbox
ensures Agent B cannot modify Agent A's context, while MxM's meta-context
protocol ensures the content is semantically meaningful.

**IP classification: Original synthesis — MxM meta-context + NemoClaw isolation.**

---

## 6. Risk Register

Risks to the synthesis that are not addressed by any source model.

### 6.1 Gate Latency Budget

**Risk:** Every tool call passes through semantic gates (Go interface calls) +
permission checks + sandbox setup. Cumulative latency could make the agent
feel sluggish.

**Mitigation:** Latency tracking per layer (LangSmith pattern). Budget: gates
must complete in <50ms; sandbox setup amortized over session (container created
at spawn, not per-tool-call).

**No source model** benchmarks gate latency. This is an omaestro-specific concern.

### 6.2 Policy Composition Complexity

**Risk:** Custom agent types that extend built-in types could produce policy
conflicts (e.g., extending `researcher` but adding `bash` tool breaks the
read-only sandbox assumption).

**Mitigation:** Type Resolver validates consistency: if a type adds a tool that
requires write access, the sandbox policy must also grant write access. Validation
errors at resolution time, not runtime.

**No source model** has type composition, so none addresses composition conflicts.

### 6.3 Multi-Provider Governance Consistency

**Risk:** Different LLM providers have different safety thresholds (Layer 1).
An action blocked by Claude might be allowed by Nemotron. If governance gates
(Layer 2) assume consistent Layer 1 behavior, switching providers could create gaps.

**Mitigation:** Layers 2-5 must be sufficient without Layer 1. Layer 1 is
defense-in-depth, not load-bearing. omaestro's governance should not depend
on any provider's built-in safety.

### 6.4 Sandbox Overhead for Simple Tasks

**Risk:** Creating a container for a quick `ls` command is disproportionate overhead.

**Mitigation:** Container-per-agent (not per-tool-call). Safe command list
(OpenCode pattern) bypasses sandbox for known-harmless operations. Container
reuse for sequential subagent spawns from the same parent.

---

## 7. Synthesis Completeness Check

Does omaestro address every P0 gap identified in each source model's gap analysis?

### Claude Code P0 Gaps
| Gap | Status | omaestro Component |
|-----|--------|--------------------|
| Runtime sandboxing | ADDRESSED | Sandbox Manager (NemoClaw) |
| Semantic tool gates | ADDRESSED | Governance Engine (MxM) |
| Append-only audit | ADDRESSED | Audit Log (MxM + LangSmith) |

### OpenCode P0 Gaps
| Gap | Status | omaestro Component |
|-----|--------|--------------------|
| Runtime sandboxing | ADDRESSED | Sandbox Manager (NemoClaw) |
| Semantic tool gates | ADDRESSED | Governance Engine (MxM) |
| Append-only audit | ADDRESSED | Audit Log (MxM + LangSmith) |
| Distributed tracing | ADDRESSED | Trace Collector (LangSmith) |

### LangSmith P0 Gaps
| Gap | Status | omaestro Component |
|-----|--------|--------------------|
| Runtime sandboxing | ADDRESSED | Sandbox Manager (NemoClaw) |
| Semantic safety gates | ADDRESSED | Governance Engine (MxM) |
| Agent management | ADDRESSED | Agent Loop Engine (OpenCode) |

### NemoClaw Gaps (implicit — no gap doc, assessed from architecture)
| Gap | Status | omaestro Component |
|-----|--------|--------------------|
| Semantic governance | ADDRESSED | Governance Engine (MxM) |
| Structured tracing | ADDRESSED | Trace Collector (LangSmith) |
| Multi-phase coordination | ADDRESSED | Agent Loop Engine + UX (Claude Code) |

**Result: All P0 gaps across all source models are addressed by the synthesis.**

---

## 8. Legal Provenance Summary

Every omaestro component traces to a license-clear source.

| Component | Source | License | Use Type |
|-----------|--------|---------|----------|
| Agent Registry | Original | Ologos | New code |
| Type Resolver | Original | Ologos | New code |
| Governance Engine | MxM framework | Ologos | Own IP |
| Sandbox Manager | NemoClaw patterns | Apache 2.0 | Derivative (attributed in NOTICE) |
| Agent Loop Engine | OpenCode patterns | MIT | Derivative (attributed in NOTICE) |
| Trace Collector | LangSmith SDK patterns | MIT | Derivative (attributed in NOTICE) |
| UX Layer | Claude Code behavioral patterns | Public interfaces | Clean-room reimplementation |
| Novel contributions (5.1-5.6) | Original synthesis | Ologos | New IP |

No proprietary source code is copied, reverse-engineered, or derived. All Apache 2.0
and MIT obligations (attribution, license notice) are met in the NOTICE file.
Claude Code patterns are derived from published documentation and operational
observation only.
