# Claude Code Safety Model

How Claude Code constrains agent behavior, and where the gaps are.

## Permission Modes

| Mode | Behavior |
|------|----------|
| Plan | Agent can only read; all writes require approval |
| AutoEdit | File edits auto-approved; Bash requires approval |
| FullAuto | All tools auto-approved (most autonomous) |

These modes control the **user approval gate** only. They do not affect:
- Hook execution (hooks always fire regardless of mode)
- Model-level safety (Anthropic's built-in refusals)
- Filesystem permissions (OS-level)

## Safety Layers (as observed)

1. **Model safety** — Claude refuses harmful requests at the model level
2. **Permission mode** — User approves/denies tool calls
3. **Hook system** — External scripts can block tool calls
4. **OS permissions** — Standard Unix user/group/other

### What's Missing

- **No network egress control** — Agent can curl any endpoint
- **No filesystem sandboxing** — Agent can read/write anywhere user can
- **No resource limits** — Agent can consume unlimited CPU/memory/disk
- **No capability dropping** — Agent runs with full user permissions
- **No audit trail** — Tool approvals are ephemeral (session-only)
- **No cross-agent isolation** — All agents share the same filesystem/network

## 4M's Current Coverage

4M adds governance via the hook system:

| Gate | What it catches | Hook type |
|------|----------------|-----------|
| Pre-commit | Secrets, API keys, large binaries in commits | PreToolUse(Bash) |
| CDAE | Bash mutations from chat context (not agent) | PreToolUse(Bash) |
| Pre-publish | Writes to protected article directories | PreToolUse(Write/Edit) |
| Force-push | Force push to main/master | PreToolUse(Bash) |

4M's coverage is **semantic** — it understands what the operation *means*.
But it's implemented as shell scripts, which means:
- A sufficiently creative prompt injection could potentially craft a command
  that passes the regex but achieves the blocked intent
- If the hook script has a bug or crashes, the tool call proceeds (fail-open)
- There's no defense against operations the hook doesn't pattern-match

## NemoClaw's Coverage (for comparison)

NemoClaw addresses the structural gaps:

| Gap | NemoClaw Solution |
|-----|------------------|
| No network control | Default-deny egress + per-binary whitelisting |
| No filesystem sandbox | Landlock LSM + container isolation |
| No resource limits | Container cgroups |
| No capability dropping | seccomp + capability bounding set |
| No audit trail | Container logs + policy violation records |
| No cross-agent isolation | Separate container per agent |

## Synthesis: Defense in Depth

```
Layer 1: Model Safety (Claude's built-in refusals)
  ↓ passes
Layer 2: 4M Semantic Gates (hook scripts check intent)
  ↓ passes
Layer 3: Permission Mode (user/operator approval)
  ↓ passes
Layer 4: NemoClaw Sandbox (OS-level enforcement)
  ↓ passes
Layer 5: Audit (all decisions logged)
```

An action must pass ALL layers. Each layer catches different things:
- Model safety catches obviously harmful requests
- 4M catches context-dependent violations (secrets in commits, wrong branch)
- Permission mode catches anything the operator doesn't want
- NemoClaw catches anything the policy doesn't allow, regardless of the above

The key property: **no single layer's failure compromises the system.**
