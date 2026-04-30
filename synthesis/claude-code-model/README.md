# Claude Code Model

**Purpose:** Document Claude Code's public interfaces, observable capabilities, and behavioral
patterns as a reference model for synthesis with NemoClaw and 4M governance.

**Legal basis:** This folder contains NO Anthropic source code. All content is derived from:
- Published documentation (docs.anthropic.com)
- Observable CLI behavior and public flags
- Published settings schema and hook system
- MCP protocol specification (open standard)
- Our own operational experience running Claude Code

Ideas, methods, interfaces, and behavioral descriptions are not copyrightable.
Only specific code expression is protected. We describe *what it does*, not *how the code does it*.

## Contents

- `capabilities.md` — Feature map and synthesis opportunities
- `interfaces.md` — Public APIs, CLI flags, settings schema, hook contracts
- `agent-model.md` — Agent spawning, lifecycle, tool dispatch, coordination patterns
- `safety-model.md` — Permission modes, tool approval, safety boundaries
- `gaps.md` — Capabilities absent from Claude Code that NemoClaw or 4M provide

## Relationship to Sibling Folders

```
internal-projects/
  nemoclaw/          # Apache 2.0 — NVIDIA's sandbox runtime (source)
  claude-code-model/ # Behavioral reference model (documentation only, no source)
```

The synthesis target is a system that combines:
1. NemoClaw's runtime isolation (Landlock, seccomp, network policy) — *code reuse OK, Apache 2.0*
2. Claude Code's agent patterns (tool dispatch, coordination, hooks) — *behavioral reimplementation*
3. 4M's governance layer (Mind, Morals, Mission, Memory) — *our own IP*
