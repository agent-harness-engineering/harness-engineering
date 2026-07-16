# omaestro — Ologos Agent Harness Architecture

The synthesis of five reference models into a unified agent governance and execution platform.

## What Is omaestro?

omaestro is not a fork of any existing system. It is a **new architecture** informed by five
independently analyzed models, each contributing a distinct capability layer:

| Model | License | Layer | Question It Answers |
|-------|---------|-------|-------------------|
| **MxM** | Ologos (ours) | Governance | *Why* should the agent act? |
| **NemoClaw** | Apache 2.0 | Enforcement | *What* can the agent physically do? |
| **OpenCode** | MIT | Agent Loop | *How* does the agent execute? |
| **Claude Code** | Public interfaces | UX Patterns | *What does mature agent UX look like?* |
| **LangSmith** | MIT (SDK) | Observability | *What happened and how well?* |

## Core Design Principle

**One agent-type declaration produces four enforcement layers automatically.**

```yaml
agent-type: researcher
  governance:   # MxM
    mind: source-protocol, confidence-signaling
    morals: [P1, P3, P7]  # applicable prohibitions
    tools: [read, grep, glob, fetch]  # permitted tools
  sandbox:      # NemoClaw
    filesystem: read-only except /tmp/workspace
    network: allow api.semanticscholar.org, deny *
    resources: { cpu: 1, memory: 2G, timeout: 30m }
  observability:  # LangSmith patterns
    trace: true
    eval-dataset: researcher-quality
    cost-budget: $0.50
  loop:         # OpenCode patterns
    provider: anthropic/claude-sonnet
    context-paths: [CLAUDE.md, project-context.md]
    auto-compact: true
```

From this single declaration, omaestro generates:
1. A **MxM gate configuration** — semantic pre-execution checks
2. A **NemoClaw sandbox policy** — OS-level isolation
3. A **trace subscription** — LangSmith-pattern observability
4. An **agent loop configuration** — provider, tools, context

## Architecture Overview

See [architecture.md](architecture.md) for the full system design.

## Documents

| File | Contents |
|------|----------|
| [synthesis-analysis.md](synthesis-analysis.md) | **START HERE** — Cross-cutting analysis justifying every design decision |
| [architecture.md](architecture.md) | System architecture, component interaction, data flow |
| [agent-types.md](agent-types.md) | Agent type definitions, tool allowlists, sandbox mappings |
| [defense-in-depth.md](defense-in-depth.md) | Six-layer security model with provenance for each layer |
| [integration-spec.md](integration-spec.md) | How each reference model maps to omaestro components |
| [roadmap.md](roadmap.md) | Implementation phases with deliverables |

## License

Copyright 2026 Ologos Corp. Licensed under the Apache License, Version 2.0.
See [LICENSE](LICENSE) and [NOTICE](NOTICE) for details.

## Legal Provenance

Every design decision in omaestro traces to one of:
- **Original work** (MxM framework, Ologos IP)
- **Apache 2.0 source** (NemoClaw — derivative works permitted with attribution)
- **MIT source** (OpenCode, LangSmith SDK — unrestricted use)
- **Public interface description** (Claude Code — behavioral observation, no source)

No proprietary code is copied, reverse-engineered, or derived.
