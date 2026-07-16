# OpenCode Capabilities — Synthesis Opportunities

## 1. Multi-Provider Abstraction

### What OpenCode Does
- Unified `Provider` interface: `SendMessages()`, `StreamResponse()`, `Model()`
- Concrete implementations for: Anthropic, OpenAI, Gemini, AWS Bedrock, Azure OpenAI, Groq, Copilot, OpenRouter, VertexAI, xAI, and local endpoints
- Provider selected at config time via agent configuration; each agent type can use a different model
- Streaming response events: content delta, thinking delta, tool use start/delta/stop, completion
- Automatic retry with exponential backoff (up to 8 retries)

### Synthesis Opportunity
NemoClaw already supports model routing (NVIDIA Cloud, NIM, vLLM, Ollama). OpenCode's provider
abstraction is a **Go-native implementation** of the same pattern. In a synthesized system:
- OpenCode's `Provider` interface could wrap NemoClaw's inference gateway
- MxM governance could gate provider selection (e.g., prohibit cloud providers in air-gapped mode)
- LangSmith tracing could intercept the `StreamResponse` channel for per-token observability

**Key insight: provider abstraction is the enabler for model-agnostic governance.**

---

## 2. Tool System

### What OpenCode Does
- `BaseTool` interface: `Info() ToolInfo` + `Run(ctx, ToolCall) (ToolResponse, error)`
- Built-in tools: Bash, Edit, Write, Patch, View, Glob, Grep, Ls, Fetch, Sourcegraph, Diagnostics
- MCP tool integration: any MCP server's tools are wrapped as `BaseTool` instances
- Tools receive session context via Go context values (session ID, message ID)
- Tool responses carry metadata (e.g., BashResponseMetadata with start/end times)
- Tools that modify state (Bash, Edit, Write, Patch) require permission; read-only tools (Glob, Grep, Ls, View) do not

### Synthesis Opportunity
Combine with NemoClaw: tool execution inside sandboxed containers. Combine with MxM:
semantic gates as a pre-tool-execution check. Unlike Claude Code's hook-based extensibility,
OpenCode's tool system is **Go interface-based** — extending it means implementing `BaseTool`,
not writing shell scripts.
**This enables compiled, type-safe tool gates rather than shell-script hooks.**

---

## 3. Agent Architecture (Coder + Task)

### What OpenCode Does
- Two agent types: **Coder** (full tool access) and **Task** (read-only tools)
- Coder agent tools: Bash, Edit, Write, Patch, View, Glob, Grep, Ls, Fetch, Sourcegraph, Diagnostics, Agent (subagent), plus MCP tools
- Task agent tools: Glob, Grep, Ls, Sourcegraph, View (read-only subset)
- Coder can spawn Task subagents via the `agent` tool
- Each agent gets its own session (parent-child relationship tracked in SQLite)
- Additional agent types: Title (generates session titles), Summarizer (compacts context)

### Synthesis Opportunity
The Coder/Task split maps cleanly to NemoClaw sandbox policy presets:
- **Coder** → baseline sandbox (filesystem write allowed within project, network restricted)
- **Task** → read-only sandbox (no writes, no network, no bash)

MxM governance could enforce agent type selection: the Mind module's reasoning constraints
could require Task agents for exploratory queries, reserving Coder for confirmed modifications.

**This is simpler than Claude Code's arbitrary agent types — two tiers with clear privilege boundaries.**

---

## 4. LSP Integration

### What OpenCode Does
- Launches language servers as child processes based on config (`lsp` section)
- Supports per-language configuration with command, args, and options
- LSP clients provide: diagnostics (errors/warnings), code intelligence
- Diagnostics tool exposes LSP results to the agent as a callable tool
- Edit/Write/Patch tools can trigger LSP diagnostics after file modifications
- LSP clients are injected into tools at construction time

### Synthesis Opportunity
No other reference system has native LSP integration:
- Claude Code relies on the model's own code understanding
- NemoClaw has no code intelligence layer
- LangSmith traces tool calls but doesn't understand code structure

In a synthesized system, LSP diagnostics could feed MxM's Quality Assurance gate:
**agent edits trigger LSP diagnostics → new errors block the edit → agent must fix before proceeding.**
This creates a compile-time safety net that is language-aware, not just pattern-based.

---

## 5. TUI (Terminal User Interface)

### What OpenCode Does
- Built with Bubble Tea (charmbracelet/bubbletea) — Go-native TUI framework
- Zone-based mouse support (lrstanley/bubblezone)
- Components: chat view, editor pane, session list, file diff viewer, permission dialog
- Theme system with configurable styles
- Real-time streaming display of agent responses
- Permission requests rendered as interactive dialogs in the TUI

### Synthesis Opportunity
NemoClaw has its own operator approval TUI. OpenCode's Bubble Tea TUI could serve as
the **unified operator interface** for a synthesized system:
- Permission requests (OpenCode pattern)
- Sandbox policy violations (NemoClaw pattern)
- MxM gate decisions (governance pattern)
- LangSmith trace summaries (observability pattern)

All rendered in a single, composable terminal interface.

---

## 6. Session Management & Persistence

### What OpenCode Does
- Sessions stored in SQLite (via sqlc-generated queries)
- Session tracks: title, message count, token usage (prompt + completion), cost, timestamps
- Parent-child session relationships (task sessions link to parent coder session)
- Auto-compact: when token usage reaches 95% of context window, summarizes and creates new session
- Session cost rolls up from child to parent
- Full message history persisted with role, content, tool calls, and tool results

### Synthesis Opportunity
Claude Code sessions are ephemeral (no persistent state beyond files). OpenCode's SQLite-backed
sessions provide a **durable audit surface**:
- MxM's append-only audit log could be implemented as SQLite tables alongside sessions
- LangSmith traces could be stored in the same database for correlated analysis
- Cost tracking enables governance-level budget enforcement (MxM could set per-agent cost limits)

**Session persistence turns the agent from a stateless tool into an auditable actor.**

---

## 7. PubSub Event Bus

### What OpenCode Does
- Internal `pubsub.Broker[T]` generic type for typed event publishing
- Used by: agent events, session events, permission requests, message updates
- Subscribers receive typed events (not raw JSON) via Go channels
- Event types: Created, Updated, Deleted (standard CRUD) plus domain-specific types
- Enables decoupled communication between TUI, agent, session, and permission components

### Synthesis Opportunity
This is the internal wiring that enables extensibility. In a synthesized system:
- MxM gates could subscribe to agent events (pre-tool, post-tool) via the PubSub bus
- NemoClaw sandbox state changes could publish events on the same bus
- LangSmith tracing could be a subscriber that captures all events for observability

**The PubSub bus is OpenCode's equivalent of Claude Code's hook system — but typed, compiled, and in-process.**

---

## Capability Matrix: What Each System Contributes

| Capability | OpenCode | Claude Code | NemoClaw | MxM | LangSmith | Synthesis Role |
|-----------|----------|-------------|----------|-----|-----------|---------------|
| Tool dispatch | Go interfaces | Hook-gated | — | Gates | Traces | OpenCode interface + MxM gates |
| Provider abstraction | Primary | Claude-only | Routing | — | — | OpenCode + NemoClaw routing |
| OS isolation | — | — | Primary | — | — | NemoClaw provides |
| Network policy | Banned commands | — | Primary | — | — | NemoClaw provides |
| Semantic safety | — | — | — | Primary | — | MxM provides |
| Agent coordination | Coder/Task | General/Explore/Plan | — | — | — | Merged type system |
| Agent isolation | Process-level | Process-level | Container | — | — | NemoClaw provides |
| LSP intelligence | Primary | — | — | — | — | OpenCode provides |
| Permission UI | TUI dialog | CLI prompt | Operator TUI | — | — | Unified TUI |
| Session persistence | SQLite | Ephemeral | — | — | — | OpenCode provides |
| Event bus | PubSub | Hooks | — | — | Callbacks | OpenCode PubSub |
| Audit | Session logs | Ephemeral | Container logs | Append-only | Trace store | MxM + SQLite + LangSmith |
| Model flexibility | Multi-provider | Claude only | Any model | Claude (current) | Any model | OpenCode + NemoClaw |
| Cost tracking | Per-session | — | — | — | Per-run | OpenCode provides |
| Context management | Auto-compact | Auto-compress | — | — | — | Merged strategies |
