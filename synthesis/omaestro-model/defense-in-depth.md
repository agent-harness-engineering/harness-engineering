# omaestro Defense in Depth

Six independent safety layers, each with clear provenance. A breach requires
all six layers to fail simultaneously.

## The Stack

```
Layer 1: Model Safety
  Provider's built-in refusals (Anthropic, OpenAI, Google, NVIDIA)
  Provenance: Provider-native
  Bypassable by: Jailbreak prompts, model weaknesses
  |
  v passes
Layer 2: 4M Semantic Gates
  Content-aware pre-execution checks
  Provenance: 4M (Ologos original)
  Bypassable by: Gate logic error, uncovered tool pattern
  |
  v passes
Layer 3: Permission Service
  User approval for modifying actions
  Provenance: Claude Code pattern (reimplemented) + OpenCode pattern
  Bypassable by: Auto-approve mode, user error
  |
  v passes
Layer 4: Command Filtering
  Banned command list + safe command list
  Provenance: OpenCode (MIT)
  Bypassable by: Scripting language equivalents (python -c "...")
  |
  v passes
Layer 5: OS-Level Sandbox
  Landlock, seccomp, network namespaces, cgroups
  Provenance: NemoClaw (Apache 2.0)
  Bypassable by: Kernel exploit (requires root or unpatched kernel)
  |
  v passes
Layer 6: Audit Trail
  Append-only log of all gate decisions + trace spans
  Provenance: 4M audit log + LangSmith tracing patterns (MIT)
  Bypassable by: N/A (detection layer, not prevention)
```

## Why Each Layer Matters

### Layer 1 — Model Safety
The first line of defense. The model itself refuses harmful requests. But models
can be jailbroken, and different providers have different safety thresholds.
**Cannot be the only layer.**

### Layer 2 — 4M Semantic Gates
Understands *context*, not just patterns. The same `git push` command is safe on
a feature branch and dangerous on main. Pattern matching (Layer 3-4) cannot
distinguish this; semantic gates can.

Key gates:
- **Pre-commit:** inspects commit content, branch, message
- **CDAE:** catches destructive actions (rm -rf, DROP TABLE, force push)
- **Content inspection:** blocks credential exposure, sensitive data in output
- **Spawn approval:** validates agent-type appropriateness for task

Implementation: Go interfaces wrapping tool dispatch (not shell hooks).
This enables **fail-closed** behavior — if the gate crashes, the tool does not execute.

### Layer 3 — Permission Service
The user decides. Three modes (from Claude Code patterns):
- **Plan:** read-only, no approval needed
- **AutoEdit:** file modifications auto-approved, bash requires approval
- **FullAuto:** all actions auto-approved

Plus persistent allowlists/denylists per project (Claude Code settings.json pattern).

### Layer 4 — Command Filtering
OpenCode's banned command list catches the most common exfiltration vectors
(curl, wget, nc, telnet, browsers). The safe command list auto-approves
known-harmless commands (ls, git status, go test), reducing approval fatigue.

**This layer is explicitly acknowledged as bypassable** — it exists to catch
obvious cases cheaply, not as a security boundary. Layer 5 is the real enforcement.

### Layer 5 — OS-Level Sandbox
The unforgeable layer. Even if Layers 1-4 all fail:
- **Landlock LSM:** filesystem access restricted to declared paths. Agent cannot
  read ~/.ssh, /etc/shadow, or any path not in its policy.
- **seccomp:** syscall filter blocks dangerous operations (ptrace, mount, etc.)
- **Network namespace:** default-deny egress. Only whitelisted endpoints reachable.
- **cgroups:** CPU, memory, time limits. Runaway agents are killed, not tolerated.

A jailbroken model that bypasses semantic gates and gets user auto-approval still
cannot read files outside its sandbox or reach endpoints not in its whitelist.

### Layer 6 — Audit Trail
Every decision at every layer is recorded:
- Gate decisions (allow/block) with rationale (4M enforcement.log)
- Tool call traces with input/output/latency/cost (LangSmith patterns)
- Sandbox violations (NemoClaw container logs)
- Permission decisions (user approve/deny with context)

This layer doesn't prevent anything — it ensures that breaches at other layers
are **detectable, attributable, and reviewable**.

## Threat Model Coverage

| Threat | Layer 1 | Layer 2 | Layer 3 | Layer 4 | Layer 5 | Layer 6 |
|--------|---------|---------|---------|---------|---------|---------|
| Data exfiltration | Partial | Yes | Yes | Partial | **Yes** | Detect |
| Credential theft | Partial | Yes | Partial | No | **Yes** | Detect |
| Destructive file ops | Partial | **Yes** | Yes | No | Yes | Detect |
| Supply chain attack | No | Yes | Yes | No | Yes | Detect |
| Force push to main | Partial | **Yes** | Yes | No | No | Detect |
| Excessive cost | No | No | No | No | Yes (timeout) | **Detect** |
| Cross-agent escalation | No | Yes | No | No | **Yes** | Detect |
| Prompt injection | Partial | Partial | No | No | **Yes** | Detect |

**Bold** = primary defense for that threat.

## Fail Modes

| Layer | Fail Mode | Consequence |
|-------|-----------|-------------|
| Model Safety | Model jailbroken | Layer 2 catches semantic violations |
| Semantic Gates | Gate crashes | **Fail-closed** — tool does not execute (Go interface) |
| Permission Service | User auto-approves | Layers 4+5 still enforce |
| Command Filtering | Scripting bypass | Layer 5 blocks network/filesystem at OS level |
| OS Sandbox | Kernel exploit | Layer 6 records the breach for forensics |
| Audit Trail | Log corruption | Other layers still prevent; detection is delayed |
