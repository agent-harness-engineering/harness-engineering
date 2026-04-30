# Capability Gaps — What LangSmith Lacks

Capabilities present in other synthesis models but absent from LangSmith,
representing what other systems must provide.

## From NemoClaw

### 1. Runtime Sandboxing
**Gap:** LangSmith observes execution but cannot constrain it. Tracing records what
happened but does not prevent unauthorized actions.
**NemoClaw provides:** Landlock LSM, seccomp filters, container isolation — enforcement
at the OS level, not just observation.
**Synthesis value:** HIGH — observability without enforcement is forensics, not safety.

### 2. Network Egress Control
**Gap:** LangSmith can trace HTTP calls but cannot block them. A traced exfiltration
is still an exfiltration.
**NemoClaw provides:** Default-deny network policy with per-binary whitelisting.
**Synthesis value:** HIGH — tracing must be paired with enforcement to be useful for security.

### 3. Resource Limits
**Gap:** LangSmith tracks token counts and latency but cannot enforce budgets.
A run consuming 10x normal tokens is flagged after the fact, not prevented.
**NemoClaw provides:** Container cgroups for CPU, memory, and time limits.
**Synthesis value:** MEDIUM — cost attribution needs cost enforcement to be actionable.

---

## From Claude Code

### 4. Agent Management
**Gap:** LangSmith traces agents but does not manage them. No agent spawning,
lifecycle management, tool dispatch, or coordination primitives.
**Claude Code provides:** Agent types, tool allowlists, parent-child spawning,
foreground/background agents, session resumption.
**Synthesis value:** HIGH — LangSmith is the instrumentation layer, not the control plane.

### 5. Tool System
**Gap:** LangSmith can trace tool calls but does not provide a tool system.
No Read, Write, Edit, Bash, Grep, Glob equivalents. No tool schema validation.
**Claude Code provides:** Typed tools with schemas, permission-gated execution,
hook extensibility.
**Synthesis value:** HIGH — LangSmith instruments tools that other systems provide.

### 6. Permission Modes
**Gap:** No approval workflow for traced actions. LangSmith assumes all actions
are authorized; it records but does not gate.
**Claude Code provides:** Allowlist, Ask, and Deny modes with user approval flow.
**Synthesis value:** HIGH — the permission layer sits between observation and execution.

### 7. Session Continuity
**Gap:** LangSmith traces are immutable records. No session resumption, no picking up
where a previous trace left off.
**Claude Code provides:** `--resume` for continuing sessions, context preservation
across session boundaries.
**Synthesis value:** LOW — tracing doesn't need continuity; agent management does.

---

## From 4M Governance

### 8. Semantic Safety Gates
**Gap:** LangSmith evaluators run after the fact (post-hoc scoring). No pre-execution
gates that can block an action before it happens based on semantic analysis.
**4M provides:** PreToolUse hooks with content inspection, branch awareness,
context-dependent block/allow decisions.
**Synthesis value:** HIGH — evaluation and enforcement are complementary but distinct.
Post-hoc scoring cannot prevent damage; pre-execution gates can.

### 9. Deontic Governance Framework
**Gap:** LangSmith has no concept of prohibitions, obligations, or permissions as
structured governance. Evaluators score quality, not compliance with ethical rules.
**4M provides:** Explicit prohibitions (P1-P10), obligations (O1-O9), conflict
resolution hierarchy, documented rationale for each rule.
**Synthesis value:** HIGH — quality metrics and governance are orthogonal concerns.

### 10. Append-Only Audit with Rationale
**Gap:** LangSmith stores traces and feedback, but these are mutable (runs can be
deleted, projects can be removed). No guarantee of append-only immutability.
The platform also does not capture *why* a decision was made, only *what* happened.
**4M provides:** enforcement.log is append-only with decision rationale.
**Synthesis value:** MEDIUM — audit immutability is a compliance requirement that
LangSmith's platform does not structurally guarantee.

### 11. Cross-Agent Context
**Gap:** LangSmith traces are independent. No mechanism for one trace to reference
or build on another trace's context. No shared state between traced executions.
**4M provides:** Meta-context files for cross-agent state persistence.
**Synthesis value:** LOW — tracing is per-execution; coordination is a control plane concern.

---

## From OpenCode

### 12. Agent Loop Architecture
**Gap:** LangSmith can trace agent loops but provides no agent loop implementation.
No conversation management, tool result routing, or multi-turn orchestration.
**OpenCode provides:** Agent loop with conversation state, tool dispatch, provider
abstraction, and multi-turn management.
**Synthesis value:** MEDIUM — LangSmith instruments the loop; OpenCode provides it.

### 13. Provider Abstraction
**Gap:** LangSmith traces model calls but does not abstract model providers.
Tracing is model-agnostic, but there's no runtime provider switching.
**OpenCode provides:** Provider interface supporting multiple LLM backends
with runtime switching.
**Synthesis value:** LOW — LangSmith's model-agnostic tracing is already compatible
with any provider.

---

## Gap Priority for Synthesis

| Priority | Gap | Source | Why |
|----------|-----|--------|-----|
| P0 | Runtime sandboxing | NemoClaw | Observation without enforcement is insufficient |
| P0 | Semantic safety gates | 4M | Post-hoc scoring cannot prevent harm |
| P0 | Agent management | Claude Code | Tracing needs something to trace |
| P1 | Network egress control | NemoClaw | Security enforcement complements tracing |
| P1 | Tool system | Claude Code | LangSmith instruments, doesn't provide tools |
| P1 | Permission modes | Claude Code | Approval workflow between observe and execute |
| P1 | Deontic governance | 4M | Quality metrics and governance are orthogonal |
| P2 | Resource limits | NemoClaw | Cost tracking needs cost enforcement |
| P2 | Agent loop | OpenCode | Orchestration layer for traced operations |
| P2 | Append-only audit | 4M | Compliance requirement beyond platform traces |
| P3 | Cross-agent context | 4M | Coordination beyond trace boundaries |
| P3 | Provider abstraction | OpenCode | Already compatible, not blocking |
| P3 | Session continuity | Claude Code | Tracing doesn't need it |

## Summary

LangSmith is a **pure observability and evaluation** platform. It excels at answering
"what happened?" and "how well did it work?" but provides no answers to:
- "Should this action be allowed?" (4M / Claude Code)
- "Can this action physically succeed?" (NemoClaw)
- "How should the agent be managed?" (Claude Code / OpenCode)

Its synthesis value is as the **instrumentation layer** that makes all other systems
visible and measurable, not as a replacement for any of them.
