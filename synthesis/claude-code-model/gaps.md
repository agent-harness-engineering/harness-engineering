# Capability Gaps — What Claude Code Lacks

Capabilities present in NemoClaw or 4M but absent from Claude Code,
representing the synthesis opportunity.

## From NemoClaw

### 1. Runtime Sandboxing
**Gap:** Agents run with full user permissions on the host filesystem and network.
**NemoClaw provides:** Landlock LSM, seccomp filters, container filesystem isolation.
**Synthesis value:** HIGH — this is the single largest safety gap in Claude Code.

### 2. Network Egress Control
**Gap:** No way to restrict which endpoints an agent can reach.
**NemoClaw provides:** Default-deny network policy with per-binary, per-endpoint,
per-method whitelisting. Operator TUI for real-time approval of blocked requests.
**Synthesis value:** HIGH — prevents data exfiltration and unauthorized API calls.

### 3. Operator Approval TUI
**Gap:** Claude Code's approval is CLI-based (y/n prompt). No visibility into
*why* something was blocked or what the agent was trying to do.
**NemoClaw provides:** Rich terminal UI showing blocked requests with context,
allowing informed approve/deny decisions.
**Synthesis value:** MEDIUM — improves operator experience for production use.

### 4. Policy-as-Code (YAML)
**Gap:** Claude Code permissions are simple pattern strings in settings.json.
No composability, no presets, no per-binary granularity.
**NemoClaw provides:** YAML policy files with preset fragments, composable rules,
per-binary scoping, hot-reload without restart.
**Synthesis value:** MEDIUM — enables shareable, versioned security policies.

### 5. Model-Agnostic Inference Routing
**Gap:** Claude Code is locked to Anthropic's Claude models.
**NemoClaw provides:** OpenShell inference gateway supporting NVIDIA Cloud,
NIM local, vLLM, Ollama — runtime switchable.
**Synthesis value:** HIGH — enables air-gapped operation and model diversity.

### 6. Blueprint Distribution (OCI)
**Gap:** Claude Code configuration is manual (settings.json + CLAUDE.md).
**NemoClaw provides:** Versioned, SHA256-verified blueprints distributed via
OCI registry. `nemoclaw deploy` for reproducible setup.
**Synthesis value:** MEDIUM — important for team/enterprise deployment.

## From 4M

### 7. Deontic Governance Framework
**Gap:** Claude Code has no structured governance beyond permissions and model safety.
**4M provides:** Prohibitions (P1-P10), Obligations (O1-O9), Permissions, Process Gates —
each with documented *reasons* and conflict resolution hierarchy.
**Synthesis value:** HIGH — governance with rationale, not just rules.

### 8. Reasoning Constraints (Mind Module)
**Gap:** Claude Code places no constraints on *how* the agent reasons.
**4M provides:** Confidence signaling, source protocols, circularity detection,
methodology requirements (1 Thess 5:21 pattern).
**Synthesis value:** MEDIUM — reduces hallucination and reasoning errors.

### 9. Semantic Tool Gates
**Gap:** Claude Code permissions are pattern-based (`Bash(git push*)`).
They don't understand *context* — the same command might be safe or dangerous
depending on which branch, what's being pushed, or who's running it.
**4M provides:** Context-aware gates that inspect content, branch, and session
state to make informed block/allow decisions.
**Synthesis value:** HIGH — catches violations pattern matching cannot.

### 10. Append-Only Audit
**Gap:** Claude Code tool approvals are ephemeral — no persistent record.
**4M provides:** `enforcement.log` with every gate decision, timestamped,
with the triggering command and the decision rationale.
**Synthesis value:** HIGH — essential for compliance and incident review.

### 11. Cross-Agent Context (Meta-Context)
**Gap:** Claude Code agents have no shared state or awareness of each other.
**4M provides:** Meta-context files that persist cross-agent state, enabling
agents to coordinate without direct communication.
**Synthesis value:** MEDIUM — enables multi-agent workflows without message bus.

## Gap Priority for Synthesis

| Priority | Gap | Source | Why |
|----------|-----|--------|-----|
| P0 | Runtime sandboxing | NemoClaw | Safety-critical |
| P0 | Semantic tool gates | 4M | Already implemented, must preserve |
| P0 | Append-only audit | 4M | Compliance requirement |
| P1 | Network egress control | NemoClaw | Data exfiltration prevention |
| P1 | Deontic governance | 4M | Framework coherence |
| P1 | Model-agnostic routing | NemoClaw | Air-gap requirement |
| P2 | Operator TUI | NemoClaw | UX improvement |
| P2 | Policy-as-code | NemoClaw | Team deployment |
| P2 | Reasoning constraints | 4M | Quality improvement |
| P3 | Blueprint distribution | NemoClaw | Enterprise deployment |
| P3 | Cross-agent context | 4M | Advanced workflows |
