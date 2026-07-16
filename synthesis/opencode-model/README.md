# OpenCode Model

**Purpose:** Document OpenCode's public interfaces, architecture, and behavioral
patterns as a reference model for synthesis with MxM governance, NemoClaw sandboxing,
Claude Code agent patterns, and LangSmith observability.

**Legal basis:** OpenCode is MIT-licensed open-source software (opencode-ai/opencode).
All content here is derived from:
- The MIT-licensed source code on GitHub
- Published README and configuration documentation
- Observable CLI behavior and public flags
- Our own analysis of the Go source packages

MIT license permits unrestricted use, modification, and redistribution.
We document architecture, interfaces, and patterns — not vendor lock-in workarounds.

**Project status:** OpenCode has been archived; the project continued as
[Crush](https://github.com/charmbracelet/crush) under the Charm team.
The architecture and patterns documented here remain valuable for synthesis
regardless of the project's continuation status.

## Contents

- `capabilities.md` — Feature map and synthesis opportunities
- `interfaces.md` — CLI flags, configuration schema, provider abstraction, tool contracts
- `agent-model.md` — Agent loop, tool dispatch pipeline, multi-turn management
- `safety-model.md` — Permission model, command safeguards, file access patterns
- `gaps.md` — Capabilities absent from OpenCode that other models provide

## Relationship to Sibling Folders

```
internal-projects/
  nemoclaw/          # Apache 2.0 — NVIDIA's sandbox runtime (source)
  claude-code-model/ # Behavioral reference model (documentation only, no source)
  opencode-model/    # MIT-licensed reference model (source-derived)
```

The synthesis landscape now includes four reference systems:

1. **NemoClaw** — Runtime isolation (Landlock, seccomp, network policy) — *Apache 2.0*
2. **Claude Code model** — Agent management patterns (tool dispatch, hooks, coordination) — *behavioral reimplementation*
3. **OpenCode model** — Multi-provider agent loop, Go-native TUI, LSP integration — *MIT, source-derived*
4. **LangSmith model** — Observability, tracing, evaluation — *reference patterns*
5. **MxM** — Governance layer (Mind, Morals, Mission, Memory) — *our own IP*

## What OpenCode Uniquely Contributes

OpenCode's architecture offers patterns not found in the other reference systems:

| Pattern | Why It Matters |
|---------|---------------|
| Multi-provider abstraction | Runtime-switchable model backends (Anthropic, OpenAI, Gemini, Bedrock, Groq, local) |
| Go-native implementation | Single binary, no Node.js dependency, embeddable in Go toolchains |
| LSP integration | Code intelligence (diagnostics, completions) fed directly to agent context |
| SQLite session persistence | Durable conversation state with cost tracking and parent-child relationships |
| Bubble Tea TUI | Rich terminal UI framework with mouse support and zone-based layout |
| PubSub event bus | Internal event-driven architecture for decoupled component communication |
