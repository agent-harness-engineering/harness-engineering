# omaestro Decision Log

Machine-readable decision record for the omaestro agent governance platform.
Any agent (Claude, GPT, Gemini, local LLM) pointed at this repo should read this file
to understand what has been decided and why.

## Format

Each decision has: ID, date, status, choice, alternatives considered, rationale, and
references to supporting analysis.

---

## D001: Project Name

- **Date:** 2026-04-30
- **Status:** DECIDED
- **Choice:** `omaestro` (single word, no hyphen)
- **Alternatives:** `maestro-model` (crowded — RunMaestro, CSA MAESTRO, Doriandarko/maestro, maestro.is all active), `o-maestro` (hyphen ambiguity)
- **Rationale:** "o" prefix namespaces to Ologos Corp. Single word is more brandable and domain/package friendly (`omaestro.ai`, `@ologos/omaestro`). No existing project uses this name.

## D002: Implementation Language

- **Date:** 2026-04-30
- **Status:** DECIDED
- **Choice:** Go
- **Alternatives:** Python (ecosystem fit but runtime overhead, no compile-time safety), Rust (overkill for orchestration), Shell/TypeScript (fail-open by nature)
- **Rationale:** (1) Fail-closed by construction — missing gate = compile error, not silent pass. (2) Single binary distribution — no runtime deps. (3) Goroutines map to agent supervision (watching subprocesses, enforcing timeouts, streaming audit). See `synthesis-analysis.md` §2 Divergence D1.
- **Reference:** `synthesis-analysis.md` §4 Design Justification DJ1

## D003: License and Copyright

- **Date:** 2026-04-30
- **Status:** DECIDED
- **Choice:** Apache 2.0, Copyright 2026 Ologos Corp
- **Alternatives:** MIT (no patent grant), proprietary (limits adoption)
- **Rationale:** Matches NemoClaw upstream (Apache 2.0). Patent grant protects contributors and users. Permissive enough for enterprise adoption.
- **Reference:** `LICENSE`, `NOTICE`

## D004: Gate Semantics

- **Date:** 2026-04-30
- **Status:** DECIDED
- **Choice:** Fail-closed (compiled interface enforcement)
- **Alternatives:** Fail-open with logging (Claude Code, OpenCode pattern)
- **Rationale:** All four reference models default to fail-open for unknown tools/actions. This is the root cause of most agent safety gaps. Compiled Go interfaces make "forgot to implement a gate" a build failure, not a runtime vulnerability.
- **Reference:** `synthesis-analysis.md` §1 Convergence C4, §2 Divergence D1

## D005: Audit Storage

- **Date:** 2026-04-30
- **Status:** DECIDED
- **Choice:** SQLite append-only log
- **Alternatives:** Ephemeral/in-memory (Claude Code pattern), external service (LangSmith pattern), flat files
- **Rationale:** Queryable without external deps. Append-only provides tamper evidence. Embeds in single binary. Structured enough for compliance queries.
- **Reference:** `synthesis-analysis.md` §2 Divergence D2

## D006: Sandbox Strategy

- **Date:** 2026-04-30
- **Status:** DECIDED
- **Choice:** Both blocklist AND process-level isolation
- **Alternatives:** Blocklist only (Claude Code), sandbox only (NemoClaw), neither (OpenCode)
- **Rationale:** Defense in depth. Blocklist catches known-bad fast. Sandbox contains unknown-bad. Neither alone is sufficient — convergence analysis shows all single-strategy models have gaps.
- **Reference:** `synthesis-analysis.md` §2 Divergence D3, `defense-in-depth.md`

## D007: Initial Build Scope

- **Date:** 2026-04-30
- **Status:** DECIDED
- **Choice:** CLI scaffold + gate engine (phases 1-2 of roadmap)
- **Alternatives:** Start with agent launcher (practical but complex), audit pipeline (useful but not differentiating), full platform (too ambitious)
- **Rationale:** CLI gives a runnable artifact immediately (`omaestro run --type coder "fix tests"`). Gate engine is the core differentiator — what makes omaestro not "yet another agent wrapper." Everything else layers on top.
- **Reference:** `roadmap.md` phases 1-2

## D008: Core Architecture Pattern

- **Date:** 2026-04-30
- **Status:** DECIDED
- **Choice:** One agent-type YAML declaration produces four enforcement layers (governance, sandbox, observability, agent loop)
- **Alternatives:** Manual configuration per layer (standard approach), code-only config (Go structs)
- **Rationale:** This is omaestro's primary novel contribution. Reduces configuration surface from 4N to N (where N = number of agent types). Makes governance the default, not an afterthought.
- **Reference:** `synthesis-analysis.md` §5 Novel Contribution NC1, `architecture.md`

## D009: Repo Location

- **Date:** 2026-04-30
- **Status:** DECIDED
- **Choice:** `agent-harness-engineering/omaestro` (standalone repo under AHE org)
- **Alternatives:** Stay in `harness-engineering/synthesis/omaestro-model/` (analysis stays, code splits), new Ologos Corp org
- **Rationale:** Analysis docs stay in synthesis/. Go code needs its own repo with its own CI, go.mod, releases. AHE org already has the team (Tracy, Micah, Justin). User confirmed: "omaestro-harness is its own repo."

---

## D010: Model Backend Modularity

- **Date:** 2026-04-30
- **Status:** DECIDED
- **Choice:** Model-agnostic via `ModelBackend` interface. omaestro does not hardcode any LLM provider. Backends are pluggable — Claude, Codex (OpenAI), Gemini, Nemotron, local models all implement the same interface.
- **Alternatives:** Claude-only (simpler but locks in), multi-provider with adapter pattern (heavier abstraction)
- **Rationale:** User explicitly wants to tinker with Codex and other models. The gate engine, audit pipeline, and supervision logic must operate on interfaces, not concrete implementations. A `ModelBackend` interface with `Name()`, `Execute()`, `StreamEvents()` methods means adding a new provider is one file, not a refactor.
- **Reference:** D004 (gates enforce on Process regardless of backend), D008 (YAML declaration maps to enforcement, not to a specific model)

## D015: CLI Backend Modularity

- **Date:** 2026-04-30
- **Status:** DECIDED
- **Choice:** CLI-agnostic via `CLIBackend` interface. omaestro can supervise any agent CLI — Claude Code, Codex CLI, aider, custom harnesses. The subprocess being supervised is abstracted behind `Spawn()` and `Attach()` methods.
- **Alternatives:** Claude Code only (limits reach), shelling out directly (no abstraction, hard to test)
- **Rationale:** Same principle as D010. Gates and audit don't care what process they're supervising. A `CLIBackend` interface means omaestro can wrap Codex CLI tomorrow without touching the gate engine or audit pipeline. User requirement: "I might want to tinker with tying in codex or something."
- **Reference:** D010 (model modularity), D004 (fail-closed gates operate on Process interface)

---

## Decision Backlog

Decisions not yet made but anticipated:

- **D011:** Agent type schema (exact YAML structure for type declarations)
- **D012:** TUI framework (Bubble Tea confirmed in analysis, not formally decided)
- **D013:** Plugin/extension model (how third parties add gates, agent types)
- **D014:** Distribution strategy (binary releases, Homebrew, Docker, etc.)
- **D016:** Default backend implementations (which model/CLI backends ship in v0.1)
- **D017:** QA agent integration (leverage `agent-harness-engineering/qa-agent` repo for testing/validation pipeline within omaestro)
- **D018:** Security agent integration (leverage a `agent-harness-engineering/sec-agent` repo for security scanning, vulnerability detection, and policy enforcement within omaestro)
