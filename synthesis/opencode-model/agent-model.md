# OpenCode Agent Model

Architecture of OpenCode's agent loop, tool dispatch pipeline, and session management,
derived from the MIT-licensed source code.

## Agent Types

OpenCode defines four agent types, each with a distinct role and tool set:

| Agent | Role | Tools | Model Config |
|-------|------|-------|-------------|
| Coder | Primary coding assistant | All tools (Bash, Edit, Write, Patch, View, Glob, Grep, Ls, Fetch, Sourcegraph, Diagnostics, Agent, MCP) | User-configurable |
| Task | Read-only subagent | Glob, Grep, Ls, Sourcegraph, View | User-configurable |
| Title | Session title generation | None (text generation only) | User-configurable |
| Summarizer | Context compaction | None (text generation only) | User-configurable |

Key property: **each agent type can use a different model from a different provider.**
The Coder might use Claude, the Task agent might use Gemini, and the Summarizer might use a local model.

## Agent Loop

The core loop in `agent.Run()`:

```
User sends message
  → Message persisted to SQLite
    → System prompt assembled (context paths + tool descriptions)
      → Provider.StreamResponse() called with message history + tools
        → Response streamed to TUI via PubSub
          → If response contains tool calls:
            → For each tool call:
              → Permission check (if tool requires it)
                → [User denies? → tool returns error, loop continues]
                → Tool.Run() executes
                  → Tool result persisted as message
                    → Loop: send updated history back to provider
          → If response is text only:
            → Message persisted, loop ends
```

### Multi-Turn Conversation Management
- Full message history sent to the provider on each turn
- Messages stored in SQLite with role (user, assistant, tool_call, tool_result)
- Auto-compact triggers at 95% context window utilization
- Summarizer agent creates a summary, new session continues with summary as context
- Parent session retains full history; child session starts fresh with summary

### Streaming Architecture
The provider returns a `<-chan ProviderEvent` channel. Events flow:
1. **Provider** → publishes token-level events (content delta, tool use delta)
2. **Agent** → accumulates events into complete messages, handles tool dispatch
3. **PubSub broker** → distributes `AgentEvent` to all subscribers
4. **TUI** → renders streaming content in real-time

## Tool Dispatch Pipeline

```
Agent receives tool call from provider response
  → Parse tool call (name + JSON input)
    → Find matching BaseTool by name
      → [Not found? → error response to provider]
      → Tool requires permission?
        → Yes: permission.Service.Request() called
          → PubSub publishes PermissionRequest
            → TUI renders permission dialog
              → User grants/denies via TUI
                → [Denied? → ErrorPermissionDenied returned]
                → [Granted persistently? → same tool+action+path auto-approved for session]
        → No: proceed directly
      → Context enriched with session ID + message ID
        → tool.Run(ctx, toolCall) executes
          → ToolResponse returned (text/image + metadata + error flag)
            → Response persisted as tool_result message
              → History updated, next provider call includes result
```

### Permission Flow Details
- Permission requests carry: session ID, tool name, description, action, params, path
- Grant types: **single** (this call only) or **persistent** (same tool+action+path for session)
- Auto-approve mode: `permission.AutoApproveSession(sessionID)` skips all permission checks
- Permission matching: tool name + action + session ID + directory path must all match for auto-approval
- No cross-session permission persistence (permissions reset each session)

### Parallel Tool Calls
The provider can return multiple tool calls in a single response.
OpenCode processes them sequentially (each tool call waits for the previous to complete).
This differs from Claude Code, which can process independent tool calls in parallel.

## Subagent Spawning

When the Coder agent uses the `agent` tool:

```
Coder decides to spawn subagent
  → AgentTool.Run() called with prompt
    → New Task agent created (read-only tools, no permission service)
      → New task session created (linked to parent session via parent_session_id)
        → Task agent runs independently with its own provider + conversation
          → On completion: result text returned to Coder
            → Task session cost rolled up to parent session
              → Coder receives result as tool response, continues
```

### Subagent Properties
- **Stateless invocation:** each subagent spawn creates a fresh agent + session
- **No follow-up messages:** the Coder sends one prompt, gets one response (no SendMessage)
- **No lateral communication:** subagents cannot communicate with each other
- **Read-only tools only:** subagents cannot modify files or run shell commands
- **Separate model possible:** task agent can be configured to use a different/cheaper model
- **Cost tracking:** subagent costs are aggregated into the parent session

### Comparison with Claude Code Agent Spawning

| Aspect | OpenCode | Claude Code |
|--------|----------|-------------|
| Agent types | 2 (Coder + Task) | 3+ (general, explore, plan, custom) |
| Subagent tools | Fixed read-only set | Configurable per type |
| Communication | One-shot (prompt → result) | Supports SendMessage follow-ups |
| Background execution | Not supported | Supported (detached process) |
| Multi-phase coordination | Not supported | Phase-based dependency ordering |
| Session persistence | SQLite (durable) | Ephemeral (process-only) |
| Model per agent | Configurable | Inherited from parent |
| Cost tracking | Per-session rollup | Not tracked |

## Context Management

### Context Paths
OpenCode loads project context from configurable file paths:
```
.github/copilot-instructions.md
.cursorrules
.cursor/rules/
CLAUDE.md, CLAUDE.local.md
opencode.md, opencode.local.md
```

These are concatenated and injected as system prompt context, similar to Claude Code's CLAUDE.md loading.

### Auto-Compact
When token usage approaches the model's context limit (95% threshold):
1. Summarizer agent creates a condensed summary of the conversation
2. New session created with summary as initial context
3. User transparently continues in the new session
4. Original session preserved with full history

This is more aggressive than Claude Code's context compression, which compresses
in-place rather than creating a new session.

## Synthesis Design: Enhanced Agent Model

In a synthesized system, OpenCode's agent model could be extended:

```
Coder decides to spawn subagent
  → 4M governance gate: is this spawn allowed? (Mind module)
    → Agent type → tool set (OpenCode pattern)
    → Agent type → sandbox policy (NemoClaw mapping)
    → Agent type → provider selection (multi-provider)
      → NemoClaw creates container with policy
        → Agent runs inside container with LSP access
          → Tool calls checked by: permission service + 4M gates + sandbox
            → LangSmith traces all events via PubSub subscription
              → On completion: result returned, container destroyed
```

Each agent gets:
1. **Tool constraints** — what tools it has (agent type)
2. **Semantic constraints** — what it *should* do (4M governance)
3. **Structural constraints** — what it *can* do (NemoClaw sandbox)
4. **Observability** — what it *did* (LangSmith traces via PubSub)
5. **Code intelligence** — what the code *means* (LSP integration)
