# OpenCode Public Interfaces

All information below is derived from the MIT-licensed source code and published documentation.

## CLI Interface

### Invocation
```
opencode [flags]
opencode -p "non-interactive prompt"
opencode -p "prompt" -f json
```

### Flags
| Flag | Purpose |
|------|---------|
| `-p, --prompt` | Non-interactive mode: run a single prompt and exit |
| `-f, --output-format` | Output format for non-interactive mode (`text`, `json`) |
| `-d, --debug` | Enable debug logging |
| `-c, --cwd` | Set working directory |
| `-v, --version` | Print version |
| `-q, --quiet` | Suppress non-essential output |

### Relevant for Synthesis
- `-p` enables headless/scripted operation (critical for spawned subagents)
- `-f json` enables machine-readable output (pipeline integration)
- No `--permission-mode` flag — permissions are always interactive via TUI
- No `--allowedTools` flag — tool sets are fixed per agent type (Coder vs Task)

## Configuration Schema (.opencode.json)

### File Locations (merged in order)
1. `$HOME/.opencode.json` (global)
2. `$XDG_CONFIG_HOME/opencode/.opencode.json` (XDG)
3. `./.opencode.json` (project-local)

### Full Structure
```json
{
  "data": {
    "directory": ".opencode"
  },
  "providers": {
    "anthropic": { "apiKey": "...", "disabled": false },
    "openai": { "apiKey": "...", "disabled": false },
    "gemini": { "apiKey": "..." },
    "groq": { "apiKey": "..." },
    "copilot": { "disabled": false },
    "openrouter": { "apiKey": "..." },
    "bedrock": {},
    "azure": {},
    "vertexai": {},
    "local": {}
  },
  "agents": {
    "coder": { "model": "claude-3.7-sonnet", "maxTokens": 5000 },
    "task": { "model": "claude-3.7-sonnet", "maxTokens": 5000 },
    "title": { "model": "claude-3.7-sonnet", "maxTokens": 80 },
    "summarizer": { "model": "...", "maxTokens": 4096 }
  },
  "mcpServers": {
    "server-name": {
      "type": "stdio",
      "command": "path/to/server",
      "args": [],
      "env": [],
      "url": "",
      "headers": {}
    }
  },
  "lsp": {
    "go": { "disabled": false, "command": "gopls", "args": [], "options": {} },
    "typescript": { "command": "typescript-language-server", "args": ["--stdio"] }
  },
  "shell": {
    "path": "/bin/bash",
    "args": ["-l"]
  },
  "contextPaths": [
    "CLAUDE.md", "opencode.md", ".cursorrules",
    ".github/copilot-instructions.md"
  ],
  "tui": { "theme": "default" },
  "autoCompact": true,
  "debug": false,
  "debugLSP": false
}
```

### Key Differences from Claude Code

| Aspect | OpenCode | Claude Code |
|--------|----------|-------------|
| Config format | JSON (`.opencode.json`) | JSON (`settings.json`) |
| Config merge | Global → XDG → local | Global → project → session |
| Provider config | Multi-provider with API keys | Single provider (Anthropic) |
| Agent config | Per-agent model selection | Model via `--model` flag |
| Hook system | None | PreToolUse/PostToolUse/Notification |
| Permission config | None (runtime TUI only) | Allowlist/deny patterns |
| Context paths | Configurable array | Fixed CLAUDE.md hierarchy |
| MCP support | stdio + SSE transports | stdio transport |

## Provider Abstraction

### Provider Interface
```go
type Provider interface {
    SendMessages(ctx context.Context, messages []message.Message, tools []tools.BaseTool) (*ProviderResponse, error)
    StreamResponse(ctx context.Context, messages []message.Message, tools []tools.BaseTool) <-chan ProviderEvent
    Model() models.Model
}
```

### Supported Providers
| Provider | Implementation | Auth |
|----------|---------------|------|
| Anthropic | Direct API | `ANTHROPIC_API_KEY` |
| OpenAI | Direct API | `OPENAI_API_KEY` |
| Gemini | Direct API | `GEMINI_API_KEY` |
| AWS Bedrock | AWS SDK | `AWS_ACCESS_KEY_ID` + `AWS_SECRET_ACCESS_KEY` |
| Azure OpenAI | Azure SDK | `AZURE_OPENAI_ENDPOINT` + key or Entra ID |
| Groq | OpenAI-compatible | `GROQ_API_KEY` |
| Copilot | GitHub OAuth | `GITHUB_TOKEN` |
| OpenRouter | OpenAI-compatible | API key |
| VertexAI | Google Cloud | `VERTEXAI_PROJECT` + `VERTEXAI_LOCATION` |
| xAI | OpenAI-compatible | API key |
| Local | OpenAI-compatible | `LOCAL_ENDPOINT` |

### Streaming Events
```go
const (
    EventContentStart  = "content_start"
    EventToolUseStart  = "tool_use_start"
    EventToolUseDelta  = "tool_use_delta"
    EventToolUseStop   = "tool_use_stop"
    EventContentDelta  = "content_delta"
    EventThinkingDelta = "thinking_delta"
    EventContentStop   = "content_stop"
    EventComplete      = "complete"
    EventError         = "error"
    EventWarning       = "warning"
)
```

## Tool Contracts

### BaseTool Interface
```go
type BaseTool interface {
    Info() ToolInfo
    Run(ctx context.Context, params ToolCall) (ToolResponse, error)
}

type ToolInfo struct {
    Name        string
    Description string
    Parameters  map[string]any
    Required    []string
}

type ToolResponse struct {
    Type     string  // "text" or "image"
    Content  string
    Metadata string  // JSON-encoded tool-specific metadata
    IsError  bool
}
```

### Built-in Tools
| Tool | Permission Required | Description |
|------|-------------------|-------------|
| `bash` | Yes | Shell command execution with banned-command list |
| `edit` | Yes | Line-range replacement in existing files |
| `write` | Yes | Create or overwrite files |
| `patch` | Yes | Apply unified diff patches |
| `fetch` | Yes | HTTP fetch (with permission gate) |
| `view` | No | Read file contents with line range |
| `glob` | No | File pattern matching |
| `grep` | No | Content search with regex |
| `ls` | No | Directory listing |
| `sourcegraph` | No | Code search via Sourcegraph |
| `diagnostics` | No | LSP diagnostics for a file |
| `agent` | No | Spawn a read-only Task subagent |

## LSP Integration Points

### Configuration
Per-language LSP server configuration in `.opencode.json`:
```json
{
  "lsp": {
    "go": { "command": "gopls" },
    "python": { "command": "pyright-langserver", "args": ["--stdio"] }
  }
}
```

### Integration Surface
- LSP clients injected into Edit, Write, Patch, View, and Diagnostics tools
- Diagnostics tool queries LSP for file-level errors and warnings
- File modification tools can trigger diagnostic refresh
- LSP results formatted as tool responses for agent consumption

## MCP Integration

### Transport Types
- **stdio** — subprocess with stdin/stdout communication
- **SSE** — HTTP Server-Sent Events with URL and headers

### MCP Tool Wrapping
Each MCP server's tools are discovered at startup and wrapped as `BaseTool` instances.
Tool names are prefixed with the MCP server name: `{server}_{tool}`.
MCP tools that modify state go through the permission system.

## Session Persistence (SQLite)

### Schema (observable from sqlc config)
- **sessions** — id, parent_session_id, title, message_count, prompt_tokens, completion_tokens, cost, timestamps
- **messages** — id, session_id, role, content, tool calls, tool results, finish_reason, timestamps

### Session Types
| Type | Purpose | Parent |
|------|---------|--------|
| Coder | Primary conversation | None |
| Task | Subagent work session | Coder session |
| Title | Auto-generate session title | Coder session |
| Summarizer | Context compaction | Coder session |
