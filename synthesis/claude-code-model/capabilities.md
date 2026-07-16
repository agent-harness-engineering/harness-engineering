# Claude Code Capabilities — Synthesis Opportunities

## 1. Tool System

### What Claude Code Does
- Provides agents with typed tools (Read, Write, Edit, Bash, Grep, Glob, etc.)
- Each tool has a defined schema (parameters, types, required fields)
- Tool calls are intercepted by hooks before and after execution
- Tools can be allowed, denied, or require user approval per permission mode

### Synthesis Opportunity
Combine with NemoClaw: Tools execute *inside* a sandboxed container. Even if a Bash tool
call is approved by MxM governance, it still can't escape the Landlock/seccomp boundary.
**Defense in depth: semantic gate (MxM) + structural gate (NemoClaw).**

---

## 2. Hook System

### What Claude Code Does
- **PreToolUse** — fires before a tool executes; can block (exit 2) or modify
- **PostToolUse** — fires after completion; can post-process results
- **Notification** — fires on events (session start, agent completion)
- Hooks receive JSON on stdin: tool name, parameters, session context
- Hooks are shell commands defined in settings.json

### Synthesis Opportunity
MxM's current gates (pre-commit, CDAE, pre-publish) are implemented as PreToolUse hooks.
In a synthesized system, NemoClaw's policy engine could *also* be a hook provider —
translating sandbox policy violations into hook-level blocks before the tool even runs.
This would give operators NemoClaw-style TUI approval for blocked operations.

---

## 3. Agent Spawning & Coordination

### What Claude Code Does
- Agents are subprocesses with their own conversation context
- Agent types define tool allowlists (general: all tools; explore: read-only; plan: read-only)
- Parent agent sends a prompt; child agent works independently and returns a result
- Multi-phase coordination: parent can define phases with dependency ordering
- Agents can be foreground (blocking) or background (detached)

### Synthesis Opportunity
Each spawned agent could run in its own NemoClaw sandbox. Agent type determines both:
- **Tool allowlist** (from Claude Code's agent type system)
- **Sandbox policy** (from NemoClaw's policy presets)

An "explore" agent would get read-only tools AND a read-only sandbox.
A "general" agent would get full tools but still within sandbox boundaries.

**This is the key insight: agent type maps to both semantic and structural constraints.**

---

## 4. Permission Modes

### What Claude Code Does
Three modes controlling tool approval:
- **Allowlist** — pre-approved tool+pattern combos execute without asking
- **Ask** — user prompted for each tool call
- **Deny** — tool call blocked entirely

Permissions are scoped: global, project-level, or session-level.
Some tools (Read, Glob, Grep) are typically auto-allowed; others (Bash, Write) require approval.

### Synthesis Opportunity
Map permission modes to NemoClaw policy presets:
- Allowlist mode → NemoClaw baseline policy (known-safe operations)
- Ask mode → NemoClaw operator approval TUI
- Deny mode → NemoClaw deny rule

This creates a unified permission surface: one policy file governs both the LLM interface
and the OS sandbox.

---

## 5. MCP (Model Context Protocol)

### What Claude Code Does
- Connects to MCP servers that expose tools and resources
- MCP is an open protocol (not Anthropic-proprietary)
- Servers can provide: tools (callable functions), resources (data sources), prompts (templates)
- Claude Code acts as an MCP client; any MCP server can extend its capabilities

### Synthesis Opportunity
NemoClaw's sandbox policies could be exposed *as an MCP server*. The LLM would see tools like:
- `sandbox.check_policy(action, target)` — query whether an action is allowed
- `sandbox.request_approval(action, justification)` — escalate to operator
- `sandbox.status()` — current sandbox state and active restrictions

This makes the sandbox *visible to the agent*, not just an invisible wall. The agent can
reason about what it's allowed to do before attempting it, reducing blocked-action churn.

---

## 6. Memory System

### What Claude Code Does
- File-based persistent memory in `~/.claude/projects/.../memory/`
- MEMORY.md index loaded at conversation start
- Memory types: user, feedback, project, reference
- Manual save/recall (no auto-indexing, no vector search)
- Memory persists across conversations but not across machines

### Synthesis Opportunity
MxM's Memory module already extends this with meta-context logging and cross-agent state.
In a synthesized system, NemoClaw's sandbox could enforce memory isolation:
- Agent A's memory files are not readable by Agent B's sandbox
- Shared memory requires explicit cross-sandbox mount (operator-approved)
- Memory writes go through QA gate (from agent-001's pattern) before persistence

---

## 7. Context Management

### What Claude Code Does
- Automatic context compression as conversation approaches limits
- CLAUDE.md files loaded at session start (project instructions)
- Git status snapshot provided at conversation start
- Tool results can be large; system manages context window budget

### Synthesis Opportunity
NemoClaw's context is the sandbox state (mounted volumes, network policy, running processes).
A synthesized system could inject sandbox context alongside CLAUDE.md:
- Current policy preset and any temporary overrides
- Active sandbox mounts and their permissions
- Recent policy violations and operator decisions

The agent would know its own constraints, not just its instructions.

---

## Capability Matrix: What Each System Contributes

| Capability | Claude Code | NemoClaw | MxM | Synthesis Role |
|-----------|-------------|----------|-----|---------------|
| Tool dispatch | Primary | — | Gates | Claude Code pattern, MxM gates |
| OS isolation | — | Primary | — | NemoClaw provides |
| Network policy | — | Primary | — | NemoClaw provides |
| Semantic safety | — | — | Primary | MxM provides |
| Agent coordination | Primary | — | — | Claude Code pattern |
| Agent isolation | — | Primary | — | NemoClaw provides |
| Permission UI | CLI prompt | Operator TUI | — | Unified surface |
| Memory | File-based | — | Extended | MxM + sandbox isolation |
| Audit | Ephemeral | Container logs | Append-only log | MxM audit + NemoClaw logs |
| Model flexibility | Claude only | Any model | Claude (current) | NemoClaw's routing |
| Policy-as-code | settings.json | YAML policies | Shell gates | Unified policy format |
