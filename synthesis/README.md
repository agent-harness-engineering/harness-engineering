# Synthesis Landscape

A capability survey of four external reference systems, analyzed for synthesis with the 4M governance framework into a unified agent harness architecture.

## Legal Basis

| Model | License | What We Use |
|-------|---------|-------------|
| **NemoClaw** | Apache 2.0 | Full source (derivative works permitted with attribution) |
| **Claude Code** | Proprietary | Public interfaces, behavioral description only (no source code) |
| **OpenCode** | MIT | Full source (unrestricted use, modification, redistribution) |
| **LangSmith** | MIT (SDK) | SDK patterns, public documentation, observable behavior |
| **4M** | Ours | Full ownership |

## Five-Model Capability Matrix

| Capability | 4M | NemoClaw | Claude Code | OpenCode | LangSmith |
|-----------|-----|----------|-------------|----------|-----------|
| **Governance / Ethics** | Deontic framework (P/O/Permission/Gate) | - | - | - | - |
| **Reasoning Constraints** | Mind module, source protocol, circularity | - | - | - | - |
| **Runtime Sandboxing** | - | Landlock + seccomp + netns | - | - | - |
| **Network Policy** | - | Per-binary, per-endpoint, per-method | - | - | - |
| **Filesystem Isolation** | Pattern-based gates | Kernel LSM (unforgeable) | - | - | - |
| **Agent Tool Dispatch** | Hook-based gates | - | Tool approval + permission modes | Go tool registry + LSP | - |
| **Agent Coordination** | Multi-phase coordinator | - | Subagent spawning | - | - |
| **Multi-Provider Support** | Claude only | NVIDIA Cloud + NIM + vLLM | Claude only | Anthropic, OpenAI, Gemini, Bedrock, Groq, local | - |
| **Tracing / Observability** | Audit log (append-only) | Container logs | - | SQLite session logs | Hierarchical spans, cost attribution |
| **Evaluation** | 33-test gate suite | vitest plugin tests | - | - | Dataset-driven eval, regression testing |
| **Human-in-the-Loop** | - | Operator TUI approval | User permission prompts | - | Annotation queues, feedback scoring |
| **Prompt Management** | CLAUDE.md + skills | - | CLAUDE.md | - | Versioned prompts, A/B deployment |
| **Session Persistence** | JSON + meta-context | Container state | Conversation history | SQLite with cost tracking | Run storage |
| **Air-Gap Operation** | No | Yes (NIM, vLLM, Ollama) | No | Yes (Ollama, local) | No (SaaS) |

## Unifying Abstraction

The synthesis target: **one agent-type declaration produces three enforcement layers automatically.**

```
agent-type: "researcher"
  -> 4M governance: Mind constraints + permitted tools + source protocol
  -> NemoClaw sandbox: filesystem policy + network whitelist + seccomp profile
  -> LangSmith trace: parent span + cost budget + eval dataset
```

Each model contributes a distinct, non-overlapping layer:

1. **4M** (ours) -- *Why* the agent should/shouldn't act (governance, reasoning, ethics)
2. **NemoClaw** (Apache 2.0) -- *What* the agent physically can't do (OS-level isolation)
3. **Claude Code model** (public interfaces) -- *How* a mature agent system behaves (patterns for reimplementation)
4. **OpenCode** (MIT) -- *Inspectable reference implementation* of the agent loop (legitimate source to study)
5. **LangSmith model** (MIT SDK) -- *Observability and quality measurement* (tracing, eval, feedback)

## Folder Contents

```
synthesis/
  README.md              <- this file
  claude-code-model/     <- agent management patterns (behavioral, no source)
  nemoclaw-model/        <- runtime sandboxing (Apache 2.0, full source)
  opencode-model/        <- open-source agent-tool dispatch (MIT, source-derived)
  langsmith-model/       <- observability and evaluation patterns (MIT SDK)
```

## Status

- [x] Claude Code model -- capabilities, interfaces, agent model, safety model, gaps
- [x] NemoClaw model -- full Apache 2.0 source + documentation
- [x] OpenCode model -- capabilities, interfaces (agent-model, safety-model, gaps pending)
- [x] LangSmith model -- capabilities, interfaces, observability model (eval-model, gaps pending)
- [ ] Synthesis architecture document -- the actual unified design
- [ ] 4M integration spec -- how governance maps to sandbox policies and traces
