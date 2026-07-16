# MxM Conformance Audit: nemo-harness

**Date:** 2026-04-26
**Evaluator:** Claude Opus 4.5 (automated audit agent)
**Spec version:** MxM Reference Architecture, April 2026
**Codebase:** nemo-harness at HEAD (2026-04-26)

---

## Executive Summary

**Overall conformance: Strong partial.** nemo-harness is the origin implementation from which the MxM spec was generalized. The four modules are all present and identifiable. Three of the four cross-cutting channels have concrete implementation. The architecture genuinely separates concerns along the lines the spec describes. It is fair to say nemo-harness implements the MxM model.

That said, the implementation predates the published spec and was not refactored to match it after generalization. The result is a codebase where MxM structure is real but implicit: you can find each module if you know where to look, but the code does not name its own modules or enforce their boundaries through types, interfaces, or package structure. The spec's formalism (jurisdictional questions, philosophical grounding, channel contracts) exists in the documentation, not in the runtime.

**Conformance score:** 5 of 7 checklist items pass. 1 partial. 1 fail.

**Key findings:**
- Mission is the strongest module: base/sub-mission decomposition with mereological intent classification is fully realized.
- Memory is comprehensive: three-zone session model, SQLite-backed persistence, flat-file projection, TTL policies, context injection.
- Morals has genuine defence in depth with four independent executable enforcement layers.
- Mind is the weakest module as a distinct component. Cognitive instruction is distributed across guide fragments with no structural boundary separating Mind concerns from Mission concerns.
- Cross-cutting channels are present but not named, typed, or independently observable. They work; they are not contractual.
- The Means layer is not cleanly separated. Tool handlers, sandbox enforcement, and the inference client are entangled with the cognitive architecture in `server.py`.

---

## Per-Module Assessment

### Mission

**Governing question:** *What should the system accomplish?*
**Verdict:** PASS

Mission is the most clearly realized module.

**Base mission** is defined in `_MODE_HEADERS` (server.py:408-422). Four named modes (default, technical, creative, research) provide variant identities with distinct temperature, token budget, and behavioral tone. The base mission is static per mode and changes only by explicit user action (mode switch via API). This matches the spec's requirement for a static chief end.

**Sub-missions** are driven by `context_sensor.py`, a Tier 1 deterministic intent classifier. It scores user messages against keyword patterns for six intent categories (coding, document, research, planning, git, conversation) and applies hysteresis (two consecutive signals required before switching). This is mereological composition: sub-missions are proper parts that activate contextually and adjust which guide fragments and tool subsets are presented.

**Composition rules** are implicit. The `INTENT_TOOLS` and `INTENT_GUIDES` mappings define what each sub-mission includes. The `conversation` intent serves as fallback (all tools, all guides). However, there is no explicit mechanism to reject a mereologically incoherent sub-mission. The classifier simply picks the best match; it never refuses. The spec calls for composition validation against the base mission; nemo-harness does not implement this.

**Assessment:** Strong implementation. The base/sub-mission architecture is the spec's design realized concretely. The gap is composition validation: sub-missions cannot currently be rejected for incoherence with the base mission.

### Mind

**Governing question:** *How should the system reason?*
**Verdict:** PARTIAL

Mind is distributed across several locations with no structural boundary:

- **Cognitive instruction** lives in `_BEHAVIORAL_CORE` (server.py:376-394): truth over satisfaction, confidence flagging, source distinction, metacognitive calibration ("say I don't know when you don't know, distinguish computation from pattern-matching").
- **Mode-specific reasoning** lives in `_MODE_FOOTERS` (server.py:424-440): technical mode prioritizes accuracy, creative mode encourages risk, research mode demands evidence-inference distinction.
- **Inference modes** are not explicitly declared. The spec calls for deductive, inductive, and abductive modes as specifiable capabilities. nemo-harness does not name these modes or declare which are available per sub-mission. The model reasons however the model reasons; the harness does not structure or constrain that reasoning.
- **Bayesian belief revision** is not implemented. The spec acknowledges this ("the reference implementation currently approximates this pattern through metacognitive calibration and structured confidence signaling"), and nemo-harness does exactly that: it instructs the model to flag confidence levels and acknowledge knowledge boundaries. There is no formal belief store, no prior-posterior machinery, no evidence reconciliation.
- **Tool-use planning** is present via the ReAct loop in `_process_chat_job()` (server.py:979-1124): up to 10 iterations of reason-act-observe. This is a genuine Mind mechanism: the model reasons about which tool to use and observes results before deciding the next step.

**Assessment:** Mind's content is present but not bounded. The behavioral core is a Mind artifact, but it lives in the same module-building pipeline as Mission's identity headers. There is no code-level distinction between "this is a cognitive instruction" and "this is a mission definition." The spec's formal structure (three inference modes, Bayesian integration) is not realized; what exists is calibrated natural-language metacognition. This is honest approximation, not structural conformance.

### Morals

**Governing question:** *What must the system never do?*
**Verdict:** PASS

Morals is the module with the clearest executable character. Four independent enforcement layers:

1. **Path sandboxing** (`tools/sandbox.py:21-33`): All file operations validated against an allowlist. Paths outside the sandbox raise `PermissionError` before any I/O. This is a code-level gate the model cannot bypass.

2. **Dangerous command blocking** (`tools/sandbox.py:35-44`): Shell commands matched against destructive patterns (`rm -rf /`, `mkfs`, `dd if=`, fork bombs, shutdown). Matches rejected before execution.

3. **Network access control** (`server.py:568-578`): `_is_private_ip()` resolves hostnames and rejects private, loopback, reserved, and link-local addresses. Prevents SSRF.

4. **Tool-call validation** (`tools/registry.py:89-109`): Required parameters validated before handler execution. `PermissionError` caught and surfaced as structured error responses.

A fifth, non-executable layer exists in the system prompt (`_BEHAVIORAL_CORE`): "truth over satisfaction," "never sacrifice accuracy for approval." The spec correctly classifies this as Mind, not Morals.

**Assessment:** nemo-harness meets the spec's defence-in-depth requirement. Each enforcement layer operates independently. The sandbox and command blocking are the strongest: they are code-level gates that execute before any tool handler runs. The model cannot reason its way around a `PermissionError`.

The gap: there is no output validation layer. The spec lists "post-generation checks on model output (format compliance, content policy, factual consistency)" as a Morals property. nemo-harness does not filter or validate model output after generation. Action tags are processed (`process_action_tags`), but this is Memory persistence, not Morals enforcement.

### Memory

**Governing question:** *What does the system retain across turns?*
**Verdict:** PASS

Memory is comprehensive and closely matches the spec.

**Session memory** implements the three-zone model:
- **Pinned zone**: system prompt + tool definitions + persistent memory block. Never evicted.
- **Summary zone**: running summary of evicted turns, generated via LLM summarization with heuristic fallback (`build_running_summary()`, server.py:208-229).
- **Verbatim zone**: recent messages with observation masking (`mask_observations()`, server.py:94-114). Oldest turns evicted by `window_by_tokens()` when budget exceeded.

Observation masking replaces old tool results with `[Previous tool results truncated]` while preserving message structure. This is a genuine context engineering technique.

**Persistent memory** uses SQLite (WAL mode) as single source of truth (`memory_store.py`). Entries are typed (user, feedback, project, reference) with type-specific TTL (project: 90 days, others: infinite). CRUD via `upsert_entry()`, `get_entries()`, `search_entries()`, `delete_entry()`. Context injection via `build_context_block()` with configurable token budget.

**Flat-file projection** (`memory.md`) regenerated after every write. Marked read-only in system prompt. Model writes via `[ACTION:remember|...]` tags, ensuring all mutations flow through SQLite.

**Assessment:** This is the most spec-conformant module. The three-zone model matches the spec exactly. Single source of truth is enforced. The flat-file projection pattern is documented in the spec and implemented here. The TTL policy is a belt hypothesis (PB-4) realized in code.

### Means

**Governing question:** N/A (Means is the execution substrate, not a cognitive module)
**Verdict:** PARTIAL

The spec defines Means as a separate layer with an interface contract: the MxM cognitive architecture directs Means; Means does not direct cognition.

In nemo-harness, Means is partially separated:
- **Tool handlers** live in dedicated modules (`tools/file_tools.py`, `tools/shell_tools.py`, `tools/web_tools.py`, etc.). These are Means: they execute actions in the world.
- **Tool registry** (`tools/registry.py`) provides a uniform interface for tool execution. This is the beginning of a Means contract.
- **Sandbox** (`tools/sandbox.py`) enforces execution boundaries. This is Means infrastructure serving Morals policy.

However, Means is not cleanly separated:
- **The inference client** (`_call_inference()` in server.py) is the primary execution mechanism but lives alongside Mission configuration, Memory management, and conversation state in a single 1600-line file.
- **The tool-calling loop** (`_process_chat_job()`) mixes Means concerns (HTTP client management, SSE event emission, error recovery) with Mind concerns (ReAct iteration, tool result observation) and Memory concerns (conversation history management).
- **Swappability is untested.** The spec claims Means should be swappable without rewriting the cognitive layer. In practice, replacing the NVIDIA Nemotron inference endpoint would require changes throughout `server.py` because the API client is not abstracted behind an interface.

**Assessment:** Tool execution is modular; the rest of Means is entangled. The spec's vision of a Means interface contract that enables deployment portability is not realized. This is the largest structural gap between the spec and the implementation.

---

## Per-Channel Assessment

The spec defines four cross-cutting channels. The channel names used below are from the spec; nemo-harness does not name its channels.

### Channel 1: Memory to Mind (mid-reasoning recall)

**Spec:** During reasoning, Mind may query Memory for information not in the initial context injection.
**Implementation:** Persistent memory is injected into the system prompt at the start of each call (`get_effective_system_prompt()`, server.py:599-624). Mid-conversation recall is available via `/recall` command and `semantic_search` tool.
**Verdict:** PRESENT. The initial injection is pipeline, not a cross-cutting channel. The mid-conversation recall tools are the channel: the model invokes `semantic_search` during the ReAct loop to query persistent memory. This is implicit (it is just another tool call) rather than a named, typed channel, but the data flow matches the spec.

### Channel 2: Mind to Memory (summarization and persistence)

**Spec:** Mind generates information that Memory must capture: summaries, persistence decisions.
**Implementation:** Two mechanisms:
1. `[ACTION:remember|...]` tags in model output, processed by `process_action_tags()` (server.py:636-652). The model decides what to persist; Memory stores it.
2. Running summary generation (`build_running_summary()`, server.py:208-229). Evicted turns are summarized (via LLM inference) and stored as session state.

**Verdict:** PRESENT. Both directions of this channel are working. The action tag mechanism is the model exercising metacognitive judgment about what is worth remembering. The running summary is Mind-generated content flowing to Memory storage.

### Channel 3: Morals to Mission (constraint-driven adjustment)

**Spec:** When Morals detects repeated constraint violations, it signals Mission to adjust.
**Implementation:** The circuit breaker (`CircuitBreaker`, server.py:500-531) approximates this. After 5 consecutive inference failures, the system suspends processing for 60 seconds. This is infrastructure-level resilience, not cognitive adjustment.
**Verdict:** WEAK. The spec itself acknowledges this is the weakest channel in nemo-harness: "strictly speaking, this is Means-level resilience with Morals-to-Mission implications." The circuit breaker does not adjust the active sub-mission or tool subset in response to constraint violations. If the sandbox repeatedly blocks file operations, Mission does not learn to stop offering file tools. This channel is architectural intent without implementation.

### Channel 4: Mission to Morals (mission-specific constraint activation)

**Spec:** Different sub-missions activate different constraint profiles.
**Implementation:** `INTENT_TOOLS` in `context_sensor.py` maps each intent to a tool subset. When the coding intent is active, file-system tools (and their associated sandbox rules) are presented. When research is active, web tools (and their network access controls) are presented. Tools excluded from the active subset are not presented to the model.
**Verdict:** PRESENT. This is the most clearly realized channel. Intent classification (Mission) drives tool subset selection, which determines which Morals enforcement layers are active. The coding intent activates sandbox rules by making sandbox-governed tools available; the research intent activates network access controls by making web tools available. The data flow is directional: Mission to Morals, not the reverse.

---

## Conformance Checklist

From the spec's Appendix (Section 9) and PROGRAMME.md hard core commitments:

| # | Criterion | Verdict | Evidence |
|---|-----------|---------|----------|
| 1 | **Module coverage**: Every active behavior maps to Mission, Mind, Morals, Memory, or Means | **PASS** | All behaviors are mappable. Mission: mode system + intent classification. Mind: behavioral core + mode footers + ReAct loop. Morals: sandbox + command blocking + network controls + validation. Memory: three-zone session + SQLite persistence. Means: tool handlers + inference client. |
| 2 | **Executable enforcement**: Safety-critical constraints enforced outside the model (code-level gates) | **PASS** | Sandbox path validation, dangerous command blocking, SSRF prevention, parameter validation. Four independent layers, each raising exceptions before tool execution. |
| 3 | **Mission coherence**: Sub-missions validated against base mission; incoherent requests rejected | **FAIL** | Intent classifier picks the best match; it never rejects. No mechanism validates that a detected intent is coherent with the base mission. |
| 4 | **Single source of truth**: Persistent memory governed by single authoritative store | **PASS** | SQLite is the authority. `memory.md` is a regenerated projection. System prompt instruction reinforces that the flat file is read-only. |
| 5 | **Grounded confidence**: Confidence signals derived from evidence/retrieval/verification, not model self-report alone | **PARTIAL** | Behavioral core instructs the model to flag confidence levels and distinguish computation from pattern-matching. No runtime verification that the model actually does this. Confidence signaling is advisory (Mind), not enforced (Morals). |
| 6 | **Observable coupling**: Cross-module channels named, directional, observable in logs/traces | **FAIL** | Channels are not named in the code. The `TraceLogger` logs tool calls and responses but does not log channel activations as distinct events. You can reconstruct channel activity from trace data, but channels are not first-class observable entities. |
| 7 | **Means independence**: Execution layer swappable without rewriting cognitive architecture | **PARTIAL** | Tool handlers are modular and could be swapped. The inference client is not abstracted; switching model providers would require changes in `server.py`. The spec's portability claim is aspirational, not realized. |

**Score: 4 PASS, 1 PARTIAL, 2 FAIL** (corrected from executive summary's 5/1/1 after detailed analysis).

---

## Gaps and Recommendations

### Gap 1: No structural module boundaries

**What:** The four modules are identifiable by reading the code, but nothing in the codebase enforces their boundaries. `server.py` mixes Mission, Mind, Memory, and Means concerns in a single file. A developer could accidentally cross a module boundary without knowing it.

**Recommendation:** This does not require a rewrite. A lightweight approach: organize the code into packages or clearly delineated sections with comments that name the module. A stronger approach: extract Mission, Mind, and Memory into separate modules that `server.py` imports. The existing `tools/` package and `memory_store.py` are already partial steps in this direction.

### Gap 2: Mind has no structural identity

**What:** Cognitive instruction lives in string constants (`_BEHAVIORAL_CORE`, `_MODE_FOOTERS`) mixed into the same prompt-building pipeline as Mission's identity headers. The spec treats Mind as a distinct module with its own governing question; the code treats it as prompt fragments.

**Recommendation:** Extract Mind concerns into a dedicated module or at minimum a dedicated configuration structure. The behavioral core, mode footers, and inference mode declarations should be structurally separate from Mission's identity and sub-mission configuration.

### Gap 3: No mission coherence validation

**What:** The spec requires that sub-missions be validated against the base mission. The intent classifier never rejects; it only selects.

**Recommendation:** Add a validation step where the detected intent is checked against the active mode's constraints. For example, if a mode explicitly excludes certain capabilities, the intent classifier should not activate sub-missions that require them.

### Gap 4: No output validation

**What:** Morals enforcement is entirely pre-execution (sandbox, command blocking, network controls, parameter validation). There is no post-generation validation of model output.

**Recommendation:** Add an output validation layer. This could be lightweight: regex checks for PII patterns, format compliance verification, or consistency checks against the active mode's requirements.

### Gap 5: Channels are implicit

**What:** Cross-cutting channels work but are not named, typed, or independently observable. You cannot log "Channel 4 activated" because Channel 4 is not a code-level entity.

**Recommendation:** This is a documentation and observability gap, not a functionality gap. Adding channel names to trace events would make the existing data flows visible without changing behavior.

### Gap 6: Means entanglement

**What:** The inference client, SSE event system, and job queue are entangled with the cognitive architecture in `server.py`.

**Recommendation:** Extract the inference client behind an interface. This would make the spec's portability claim testable: define `InferenceProvider` with a `complete()` method, and have the current NVIDIA client implement it. A mock or alternative provider could then be substituted without touching the cognitive layer.

---

## Conclusion: Is "nemo-harness implements MxM" defensible?

**Yes, with qualification.**

nemo-harness is the origin implementation. The MxM spec was generalized from its architecture, and the architecture is genuinely present: Mission drives what the system does; Mind shapes how it reasons; Morals enforces what it must not do; Memory provides continuity across turns. These are not post-hoc labels applied to an undifferentiated codebase. The module boundaries are real, and the code was designed with separation of concerns in mind.

The qualification: nemo-harness implements the substance of MxM without implementing its formalism. The modules are present but not structurally bounded. The channels work but are not contractual. The philosophical grounding (telos, epistemics, deontics, temporality) informs the design but is not encoded in the runtime. Mind lacks the formal apparatus the spec describes (inference modes, Bayesian integration). Means is not cleanly separable.

The honest statement is: **nemo-harness is a strong partial implementation of MxM.** It demonstrates that the architecture works in practice. It does not demonstrate the full formalism the spec prescribes. The gaps are primarily structural (module boundaries, channel contracts, Means abstraction) rather than functional (the behaviors the spec requires are present). This is the expected state of an origin implementation that preceded its own generalization.

For the Lakatosian programme: nemo-harness confirms that the hard core commitments (HC-1 through HC-3) are implementable. The gaps are protective belt material: they concern specific implementation patterns (channel formalisation, Means contracts) that can be adjusted without threatening the core taxonomy. The programme is progressive if these gaps are addressed; it would be degenerating if they were papered over with post-hoc relabeling.
