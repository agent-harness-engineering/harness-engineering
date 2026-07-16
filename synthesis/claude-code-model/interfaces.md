# Claude Code Public Interfaces

All information below is derived from published documentation and observable behavior.

## CLI Interface

### Invocation
```
claude [options] [prompt]
claude --model <model-id>
claude --allowedTools <tool-list>
claude --permission-mode <mode>
claude --resume <session-id>
claude -p "non-interactive prompt"
```

### Relevant Flags for Synthesis
- `--allowedTools` — restrict which tools the agent can use (maps to agent type)
- `--permission-mode` — plan|autoEdit|fullAuto (escalating autonomy)
- `--model` — model selection at invocation time
- `-p` — non-interactive mode (critical for spawned agents)
- `--resume` — session continuation (enables multi-turn agents)

## Settings Schema (settings.json)

### Hook Configuration
```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash|Write|Edit",
        "hooks": [
          {
            "type": "command",
            "command": "/path/to/gate-script.sh"
          }
        ]
      }
    ],
    "PostToolUse": [...],
    "Notification": [...]
  }
}
```

### Hook Input Contract (stdin JSON)
```json
{
  "tool_name": "Bash",
  "tool_input": {
    "command": "git push origin main"
  },
  "session_id": "...",
  "agent_id": "..."
}
```

### Hook Output Contract
- Exit 0 — allow (optionally modify via stdout JSON)
- Exit 2 — block (stderr message shown to agent)
- Other exit codes — treated as errors

### Permission Configuration
```json
{
  "permissions": {
    "allow": [
      "Bash(npm test)",
      "Bash(git status)",
      "Read",
      "Glob",
      "Grep"
    ],
    "deny": [
      "Bash(rm -rf *)"
    ]
  }
}
```

Pattern format: `ToolName(pattern)` where pattern is matched against tool input.

## MCP Server Configuration
```json
{
  "mcpServers": {
    "server-name": {
      "command": "node",
      "args": ["path/to/server.js"],
      "env": { "KEY": "value" }
    }
  }
}
```

MCP servers extend the tool set. Any MCP-compatible server can provide additional tools.

## CLAUDE.md Contract

Project instructions loaded at session start. Searched in:
1. Current working directory
2. Parent directories (up to git root)
3. `~/.claude/` (global)

Content is injected into the system prompt. This is the primary mechanism for
project-level governance — MxM's entire framework operates through this interface.

## Agent Spawn Interface (Behavioral)

When Claude Code spawns a subagent, the observable contract is:
1. Parent provides: prompt text, agent type (determines tool allowlist)
2. Child receives: prompt + relevant context (CLAUDE.md, working directory)
3. Child executes independently with its own tool access
4. Child returns: text result to parent
5. Parent can continue or spawn additional agents

Agent types and their tool access:
- `general-purpose` — all tools available
- `Explore` — read-only tools (Read, Glob, Grep, Bash read-only)
- `Plan` — read-only tools, architecture focus

## Session Lifecycle

1. **Start** — CLAUDE.md loaded, git status captured, hooks initialized
2. **Active** — tool calls dispatched, hooks fire, user approves/denies
3. **Compress** — context automatically compressed as window fills
4. **End** — session can be resumed with `--resume`

No persistent state beyond: memory files, git changes, filesystem changes.
