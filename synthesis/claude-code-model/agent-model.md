# Claude Code Agent Model

Behavioral description of how Claude Code manages agents, derived from
published documentation and operational experience.

## Spawning

### Single Agent
- Parent formulates a task prompt
- Agent type selected (general, explore, plan)
- Child process launched with its own conversation context
- Child inherits: working directory, CLAUDE.md instructions, relevant context
- Child does NOT inherit: parent's full conversation history

### Multi-Phase Coordination
- Parent defines phases with dependency ordering
- Phase N+1 agents only spawn after Phase N agents all complete
- If any agent in a phase fails, dependent phases do not spawn
- Coordinator tracks status and reports results

### Background Agents
- Spawned as detached processes
- Notify on completion (via hook system)
- Results retrievable after completion
- No interactive approval — must run in permissive mode

## Tool Dispatch

Each tool call follows this pipeline:

```
Agent decides to use tool
  → Permission check (allowlist/ask/deny)
    → PreToolUse hooks fire
      → [Hook blocks? → agent gets error message, adjusts]
      → Tool executes
        → PostToolUse hooks fire
          → Result returned to agent
```

Key property: hooks are the **only** extensibility point in this pipeline.
MxM governance operates entirely through this hook interface.

## Agent Isolation (Current)

Claude Code agents have **process-level** isolation only:
- Separate OS process
- Own conversation context (not shared with parent)
- Same filesystem access as parent
- Same network access as parent
- Same user permissions as parent

There is NO:
- Filesystem isolation between agents
- Network isolation between agents
- Resource limits (CPU, memory, time) per agent
- Capability restriction beyond tool allowlist

**This is the primary gap that NemoClaw fills.**

## Agent Communication

- **Parent → Child:** Initial prompt (one-way at spawn)
- **Child → Parent:** Final result text (one-way at completion)
- **Child → Child:** None (no lateral communication)
- **SendMessage:** Parent can send follow-up messages to a running foreground agent

No shared memory, no message bus, no event system between agents.

## Synthesis Design: Sandboxed Agent Model

In a synthesized system, the spawn pipeline becomes:

```
Agent decides to spawn child
  → MxM governance gate: is this spawn allowed?
    → Agent type → tool allowlist (Claude Code pattern)
    → Agent type → sandbox policy preset (NemoClaw mapping)
      → NemoClaw creates container with policy
        → Child runs inside container
          → Tool calls go through hooks (MxM) AND sandbox (NemoClaw)
            → On completion: container destroyed, result returned to parent
```

Each agent gets:
1. **Semantic constraints** — what it *should* do (MxM governance)
2. **Structural constraints** — what it *can* do (NemoClaw sandbox)
3. **Tool constraints** — what tools it has access to (agent type allowlist)

Three independent enforcement layers, any one of which is sufficient to prevent
unauthorized actions. All three failing simultaneously is the only breach path.
