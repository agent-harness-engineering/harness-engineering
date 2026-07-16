# OpenCode Safety Model

How OpenCode constrains agent behavior, and where the gaps are.

## Permission Model

OpenCode uses a **runtime permission service** rather than a static configuration:

### How It Works
1. Tools that modify state declare themselves as permission-requiring at construction time
2. Before executing, the tool calls `permission.Service.Request()`
3. The PubSub system delivers the request to the TUI as an interactive dialog
4. User can **Grant** (this call only), **Grant Persistently** (same tool+action+path for session), or **Deny**
5. Denied requests return `ErrorPermissionDenied` to the agent

### Permission Matching (for persistent grants)
A persistent grant matches when ALL of these are equal:
- Tool name (e.g., "bash")
- Action (e.g., "execute")
- Session ID
- Directory path

### No Pre-Configuration
Unlike Claude Code, there is **no way to pre-configure permissions** in `.opencode.json`.
Every modifying action requires runtime approval unless auto-approve is enabled for the session.
There are no allowlist patterns, no deny patterns, no permission modes.

### Auto-Approve Mode
`permission.AutoApproveSession(sessionID)` bypasses all permission checks for a session.
This is the equivalent of Claude Code's "FullAuto" mode, but it's all-or-nothing —
there is no intermediate mode like "AutoEdit" (edits approved, bash requires approval).

## Command Execution Safeguards

### Banned Commands
The Bash tool maintains a hardcoded list of banned commands:
```
alias, curl, curlie, wget, axel, aria2c,
nc, telnet, lynx, w3m, links, httpie, xh,
http-prompt, chrome, firefox, safari
```

These are **network-access and browser commands** — the primary defense against data exfiltration
and unauthorized external communication.

### Safe Read-Only Commands
Commands in the safe list bypass permission checks:
```
ls, echo, pwd, date, cal, uptime, whoami, id, groups, env, printenv, ...
git status, git log, git diff, git show, git branch, ...
go version, go help, go list, go env, go doc, go vet, go fmt, go test, go build, ...
```

### Execution Constraints
- Default timeout: 1 minute (60,000ms)
- Maximum timeout: 10 minutes (600,000ms)
- Maximum output length: 30,000 characters (truncated if exceeded)
- Persistent shell session: environment state carries between commands

## File Access Patterns

### Read Access
- View, Glob, Grep, Ls tools have unrestricted read access
- No path restrictions — can read any file the OS user can access
- No file size limits on reads

### Write Access
- Edit, Write, Patch tools require permission per invocation
- No path restrictions beyond permission approval
- No protected directory concept
- File change tracking: modifications tracked per session for diff visualization

## Comparison with Claude Code Permission Modes

| Aspect | OpenCode | Claude Code |
|--------|----------|-------------|
| Permission modes | 2 (ask all / auto-approve all) | 3 (Plan / AutoEdit / FullAuto) |
| Pre-configured rules | None | Allowlist + deny patterns in settings.json |
| Hook system | None | PreToolUse / PostToolUse / Notification |
| Network restriction | Banned command list | None (model-level only) |
| File path restrictions | None | None (OS-level only) |
| Persistent permissions | Per-session only | Per-project in settings.json |
| Permission granularity | Tool + action + path | Tool + input pattern |

## Safety Layers (as implemented)

```
Layer 1: Model Safety (provider's built-in refusals)
  ↓ passes
Layer 2: Banned Command List (hardcoded network/browser commands)
  ↓ passes
Layer 3: Safe Command List (read-only commands auto-approved)
  ↓ not in safe list
Layer 4: Permission Service (user approval via TUI dialog)
  ↓ passes
Layer 5: OS Permissions (standard Unix user/group/other)
```

### What's Present (vs Claude Code)

- **Banned command list** — OpenCode explicitly blocks network tools (curl, wget, nc, etc.)
  that Claude Code does not restrict at all. This is a stronger default posture against
  data exfiltration, though it's a blocklist (bypassable) rather than an allowlist.
- **Safe command list** — explicit auto-approval for known-safe operations, reducing
  permission fatigue without compromising safety.
- **Persistent shell** — state persistence between commands means the agent can build
  up context, but also means environment pollution is possible.

### What's Missing

- **No hook/gate system** — no extensibility point for external safety logic.
  MxM's semantic gates have no integration surface in OpenCode.
- **No filesystem sandboxing** — agent can read/write anywhere the OS user can.
- **No network egress control** — banned command list is a blocklist, not a sandbox.
  The agent could use `python -c "import urllib..."` to bypass the curl ban.
- **No resource limits** — no CPU, memory, or disk quotas per agent or session.
- **No capability dropping** — agent runs with full user permissions.
- **No audit trail** — permission decisions are in-memory only (not persisted).
  Session messages record tool calls but not approval decisions.
- **No cross-agent isolation** — Coder and Task agents share the same filesystem.
- **No content-aware gates** — permissions check tool+action+path, not the actual
  content being written or the command being executed (beyond the banned list).

## MxM Coverage Gaps in OpenCode

Because OpenCode has no hook system, MxM's current gate implementation (PreToolUse hooks)
cannot be directly ported. In a synthesized system, the integration points would be:

| MxM Gate | Claude Code Integration | OpenCode Integration (proposed) |
|---------|------------------------|-------------------------------|
| Pre-commit | PreToolUse(Bash) hook | Wrap Bash tool's `Run()` method |
| CDAE | PreToolUse(Bash) hook | PubSub subscriber on tool events |
| Pre-publish | PreToolUse(Write/Edit) hook | Wrap Write/Edit/Patch tools |
| Force-push | PreToolUse(Bash) hook | Wrap Bash tool with pattern check |
| Audit | enforcement.log file | SQLite table alongside sessions |

**The key difference: Claude Code uses shell-script hooks (fail-open, regex-based).
OpenCode's Go interfaces enable compiled, type-safe gates (fail-closed possible, structured checks).**

## NemoClaw Coverage (for comparison)

| Gap | NemoClaw Solution |
|-----|------------------|
| No filesystem sandbox | Landlock LSM + container isolation |
| No network control | Default-deny egress + per-binary whitelisting |
| Banned command bypass | Irrelevant — sandbox enforces at syscall level |
| No resource limits | Container cgroups |
| No capability dropping | seccomp + capability bounding set |
| No audit trail | Container logs + policy violation records |
| No cross-agent isolation | Separate container per agent |

## Synthesis: Defense in Depth

```
Layer 1: Model Safety (provider's built-in refusals)
  ↓ passes
Layer 2: MxM Semantic Gates (Go-interface wrappers on tools)
  ↓ passes
Layer 3: Banned Command List (OpenCode's network protection)
  ↓ passes
Layer 4: Permission Service (user approval via TUI)
  ↓ passes
Layer 5: NemoClaw Sandbox (OS-level enforcement)
  ↓ passes
Layer 6: LangSmith Audit (all decisions traced and persisted)
```

OpenCode's contribution to the defense stack: **the banned command list and safe command
list provide a middle ground between model-level safety and OS-level sandboxing.**
They catch the most common exfiltration vectors without requiring container infrastructure.
