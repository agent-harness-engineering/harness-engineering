# Capability Gaps — What OpenCode Lacks

Capabilities present in Claude Code, NemoClaw, 4M, or LangSmith but absent from OpenCode,
representing the synthesis opportunity.

## From Claude Code

### 1. Hook/Gate Extensibility
**Gap:** OpenCode has no mechanism for external scripts or plugins to intercept tool calls.
The tool dispatch pipeline is closed — tools execute directly after permission approval.
**Claude Code provides:** PreToolUse, PostToolUse, and Notification hooks with stdin JSON
contracts and exit-code-based block/allow decisions.
**Synthesis value:** HIGH — without hooks, 4M governance cannot be layered externally.
OpenCode's Go interfaces offer an alternative: compiled tool wrappers instead of shell hooks.

### 2. Background Agent Execution
**Gap:** OpenCode subagents always run synchronously (Coder blocks until Task completes).
No detached/background agent capability.
**Claude Code provides:** Background agents with notification on completion, enabling
parallel work streams.
**Synthesis value:** MEDIUM — important for long-running research tasks and multi-phase workflows.

### 3. Multi-Phase Agent Coordination
**Gap:** OpenCode supports only single-level spawning (Coder → Task). No dependency ordering,
no phase-based coordination, no multi-level agent hierarchies.
**Claude Code provides:** Phase-based coordination where Phase N+1 depends on Phase N completion.
**Synthesis value:** MEDIUM — enables complex multi-step workflows with dependency management.

### 4. Agent Follow-Up Messages (SendMessage)
**Gap:** OpenCode subagent invocations are one-shot: one prompt in, one result out.
The parent cannot send follow-up instructions to a running subagent.
**Claude Code provides:** SendMessage to running foreground agents, enabling iterative refinement.
**Synthesis value:** LOW — most subagent tasks are self-contained; follow-ups add complexity.

### 5. Configurable Permission Modes
**Gap:** OpenCode has only two modes: ask-for-everything or auto-approve-everything.
No intermediate mode where file edits are auto-approved but bash requires approval.
**Claude Code provides:** Plan (read-only), AutoEdit (edits OK, bash asks), FullAuto (all OK).
**Synthesis value:** MEDIUM — graduated autonomy is important for different trust levels.

### 6. Project-Level Permission Persistence
**Gap:** OpenCode permissions reset every session. No way to pre-configure "always allow
`npm test`" or "always deny `rm -rf`" in a project config file.
**Claude Code provides:** `permissions.allow` and `permissions.deny` in settings.json
with tool+pattern matching.
**Synthesis value:** MEDIUM — reduces permission fatigue for known-safe operations.

---

## From NemoClaw

### 7. Runtime Sandboxing
**Gap:** Agents run with full user permissions on the host filesystem and network.
The banned command list is a blocklist easily bypassed via scripting languages.
**NemoClaw provides:** Landlock LSM, seccomp filters, container filesystem isolation.
**Synthesis value:** HIGH — the banned command list is security theater without structural enforcement.

### 8. Network Egress Control
**Gap:** Banned command list blocks `curl`, `wget`, etc., but cannot prevent
`python -c "import urllib..."` or `node -e "fetch(...)"`.
**NemoClaw provides:** Default-deny network policy at the syscall level, per-binary whitelisting.
**Synthesis value:** HIGH — structural enforcement where OpenCode's blocklist fails.

### 9. Policy-as-Code (YAML)
**Gap:** OpenCode configuration is JSON-only, covering provider/model setup but not
security policies. No composable policy presets.
**NemoClaw provides:** YAML policy files with preset fragments, hot-reload, per-binary scoping.
**Synthesis value:** MEDIUM — enables shareable, versioned security policies.

### 10. Blueprint Distribution (OCI)
**Gap:** OpenCode configuration is manual (edit .opencode.json). No versioned, reproducible
deployment mechanism.
**NemoClaw provides:** OCI-distributed blueprints with SHA256 verification.
**Synthesis value:** MEDIUM — important for team/enterprise deployment.

---

## From 4M

### 11. Deontic Governance Framework
**Gap:** OpenCode has no structured governance beyond its permission service.
No prohibitions, obligations, or process gates with documented rationale.
**4M provides:** P1-P10 prohibitions, O1-O9 obligations, conflict resolution hierarchy.
**Synthesis value:** HIGH — governance with reasons, not just rules.

### 12. Semantic Tool Gates
**Gap:** OpenCode permissions check tool name + action + path, but not content.
The same `bash` command might be safe or dangerous depending on context (branch, content, state).
**4M provides:** Context-aware gates that inspect command content, branch state, and session context.
**Synthesis value:** HIGH — catches violations that tool+path matching cannot.

### 13. Append-Only Audit
**Gap:** OpenCode persists tool calls in SQLite messages but does not record permission decisions,
gate outcomes, or governance rationale. No append-only guarantee.
**4M provides:** `enforcement.log` with timestamped gate decisions and rationale.
**Synthesis value:** HIGH — essential for compliance and incident review.

### 14. Reasoning Constraints (Mind Module)
**Gap:** OpenCode places no constraints on how the agent reasons. No confidence signaling,
no source protocols, no circularity detection.
**4M provides:** Methodology requirements, confidence levels, source verification.
**Synthesis value:** MEDIUM — reduces hallucination and reasoning errors.

### 15. Cross-Agent Context (Meta-Context)
**Gap:** OpenCode agents share no state. Subagent results are returned as text;
no persistent cross-agent context mechanism.
**4M provides:** Meta-context files for cross-agent state persistence.
**Synthesis value:** MEDIUM — enables multi-agent coordination without message bus.

---

## From LangSmith

### 16. Distributed Tracing
**Gap:** OpenCode has PubSub events and SQLite persistence, but no structured tracing
with spans, parent-child relationships, and metadata annotations.
**LangSmith provides:** Hierarchical traces with input/output capture, latency,
token usage, and custom metadata per span.
**Synthesis value:** HIGH — the PubSub bus is the natural integration point for trace emission.

### 17. Evaluation Framework
**Gap:** OpenCode has no mechanism to evaluate agent performance, compare model quality,
or track regression across sessions.
**LangSmith provides:** Dataset-based evaluation with custom evaluators, scoring, and comparison.
**Synthesis value:** MEDIUM — important for model selection and governance validation.

### 18. Cost & Usage Analytics
**Gap:** OpenCode tracks per-session token counts and cost in SQLite, but provides
no aggregation, visualization, or alerting.
**LangSmith provides:** Dashboard-level analytics with cost tracking, latency distributions,
and usage patterns over time.
**Synthesis value:** MEDIUM — OpenCode's SQLite data is the raw material; LangSmith provides the lens.

---

## What OpenCode Uniquely Provides (Not Gaps)

For completeness, these capabilities are present in OpenCode but absent from one or more
other reference systems:

| Capability | Present In | Absent From |
|-----------|------------|-------------|
| Multi-provider abstraction | OpenCode | Claude Code (Claude-only) |
| LSP integration | OpenCode | Claude Code, NemoClaw, LangSmith |
| SQLite session persistence | OpenCode | Claude Code (ephemeral) |
| Go-native single binary | OpenCode | Claude Code (Node.js) |
| Bubble Tea TUI | OpenCode | Claude Code (CLI), NemoClaw (separate TUI) |
| Banned command list | OpenCode | Claude Code (no network restrictions) |
| Typed PubSub event bus | OpenCode | Claude Code (shell hooks) |
| Per-agent model selection | OpenCode | Claude Code (single model) |
| Cost rollup (parent ← child) | OpenCode | Claude Code, NemoClaw |

---

## Gap Priority for Synthesis

| Priority | Gap | Source | Why |
|----------|-----|--------|-----|
| P0 | Runtime sandboxing | NemoClaw | Banned command list is bypassable |
| P0 | Semantic tool gates | 4M | Permission service is content-blind |
| P0 | Append-only audit | 4M | Permission decisions not persisted |
| P0 | Distributed tracing | LangSmith | PubSub bus ready for integration |
| P1 | Hook/gate extensibility | Claude Code | Required for 4M integration |
| P1 | Network egress control | NemoClaw | Blocklist insufficient |
| P1 | Deontic governance | 4M | Framework coherence |
| P1 | Configurable permission modes | Claude Code | Graduated autonomy |
| P2 | Background agents | Claude Code | Long-running task support |
| P2 | Multi-phase coordination | Claude Code | Complex workflow support |
| P2 | Policy-as-code | NemoClaw | Team deployment |
| P2 | Evaluation framework | LangSmith | Model selection validation |
| P3 | Blueprint distribution | NemoClaw | Enterprise deployment |
| P3 | Cross-agent context | 4M | Advanced workflows |
| P3 | Agent follow-up messages | Claude Code | Iterative refinement |
