# Maestro Agent: Model-Agnostic Orchestration Layer

A model-agnostic orchestration layer that replicates the harness capabilities observed in Claude Code for any LLM backend. The Maestro Agent sits between the human operator and an arbitrary LLM inference endpoint, providing deterministic execution control, permission gating, context management, and governance integration regardless of which model generates the reasoning.

## Problem Statement

Claude Code demonstrates a mature harness architecture: tool interception, permission confirmation, context window management, structured output parsing, and agent isolation. However, these capabilities are tightly coupled to the Anthropic API and Claude's tool-use format. Teams deploying multiple LLM backends (GPT, Gemini, Nemotron, open-weight models) face a choice between:

1. **Vendor lock-in** -- adopting one provider's harness and accepting its constraints
2. **Fragmented governance** -- building bespoke harnesses per model with inconsistent safety properties
3. **Lowest common denominator** -- using a thin wrapper that sacrifices the governance features that make agentic execution safe

The Maestro Agent eliminates this trilemma by extracting the harness pattern into a standalone orchestration layer.

## Core Concept

The Maestro Agent implements the [4M cognitive architecture](../theory/4m-reference-architecture/index.md) (Mission, Mind, Morals, Memory) as a model-agnostic runtime. It enforces the [Agentic Execution Control Plane](../docs/agentic-execution-control-plane.md) pattern -- Agent proposes, deterministic controls constrain, humans authorize irreversible risk -- across any LLM backend.

```
                  +-------------------+
                  |   LLM Backend     |  Claude, GPT, Gemini, Nemotron, etc.
                  +--------+----------+
                           |
                  +--------v----------+
                  |  Maestro Agent    |  Model-agnostic orchestration
                  |  Orchestrator     |
                  |                   |
                  |  Tool Proxy       |  Intercepts and normalizes tool calls
                  |  Permission Gate  |  Allow / Deny / Escalate / Fail Closed
                  |  Context Manager  |  Per-model token counting + compaction
                  |  Output Validator |  Format + disposition + policy validation
                  |  Streaming Adapter|  Unified SSE across providers
                  |  Audit Store      |  Decision trace + rollback evidence
                  +--------+----------+
                           |
              +------------+------------+
              |    4M Governance Layer   |
              |  Mission | Mind | Morals | Memory
              +------------+------------+
                           |
                  +--------v----------+
                  |  Gate Scripts      |  CDAE, pre-commit, pre-push,
                  |                    |  pre-delete, pre-publish
                  +--------+----------+
                           |
                  +--------v----------+
                  |  Execution Layer   |  Read, Write, Mutate, Publish,
                  |                    |  Delete, Exfiltrate
                  +-------------------+
```

## Key Properties

- **Model-agnostic**: Supports any LLM that produces text or structured tool calls. Parsers handle OpenAI function calling, Anthropic tool use, and raw text action tags.
- **Governance-first**: 4M modules (Mission, Mind, Morals, Memory) wrap all operations. Safety constraints are executable code, not prompt suggestions.
- **Fail-closed**: Permission gates default to deny on timeout. Self-exemption is prevented by read-only config boundaries.
- **Context-aware**: Per-model token counting with pre-compaction triggers. Sliding window preserves system prompt + persistent memory + recent turns.
- **Risk-classified**: Execution capabilities are tiered (Read/Write/Mutate/Publish/Delete/Exfiltrate) with tier-appropriate gate actions and audit verbosity.
- **Auditable**: All governance decisions, tool calls, approvals, and execution results are logged to an immutable Audit & Provenance Store.
- **Portable**: Same governance spec drives local, cloud, and air-gapped deployments via the Means interface contract.

## Documents

| Document | Purpose |
|----------|---------|
| [architecture-spec.md](architecture-spec.md) | Full architecture specification with component design |
| [gap-analysis.md](gap-analysis.md) | Claude Code vs Maestro Agent capability gap analysis |
| [systems-diagram.svg](systems-diagram.svg) | Systems-level architecture diagram |
| [priority-roadmap.md](priority-roadmap.md) | Implementation priority roadmap (P0-P3) |

## Relationship to Other Work

- **4M Reference Architecture** (`theory/4m-reference-architecture/`): The cognitive model the Maestro Agent implements
- **AECP Whitepaper** (`docs/agentic-execution-control-plane.md`): The execution control plane the Maestro Agent enforces
- **Harness Engineering Program** (`PROGRAMME.md`): The Lakatosian research program this work advances
- **Agent Implementations** (`agents/`): Model-specific instances governed by the Maestro Agent pattern

## Status

Draft specification. Derived from empirical analysis of Claude Code's harness architecture and gap analysis against model-agnostic requirements.
