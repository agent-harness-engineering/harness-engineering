# Harness Engineering: A Lakatosian Research Programme

Lakatos (1978) argues that science progresses through research programmes, not isolated theories. A programme has an **irrefutable hard core** (protected by methodological decision), a **protective belt** of auxiliary hypotheses (adjusted to absorb anomalies), a **positive heuristic** (directing where to push), and a **negative heuristic** (directing what not to touch). A programme is **progressive** when its belt modifications yield novel predictions that are confirmed; it is **degenerating** when modifications are ad hoc patches that save the core without generating new content.

This document frames harness engineering as such a programme.

---

## 1. Hard Core

The hard core is irrefutable by methodological decision. Anomalies are absorbed by the protective belt, not by weakening these commitments.

**HC-1. Exhaustive cognitive taxonomy.** The MxM taxonomy (Mission, Mind, Morals, Memory) is exhaustive and jointly sufficient for harness cognitive architecture. Every cognitive concern in a well-formed harness maps to exactly one of these four modules. If a concern does not fit, the belt is adjusted (the module's scope is clarified, the concern is reclassified), not the taxonomy.

**HC-2. Categorical distinction between cognition and execution.** The cognitive architecture (MxM) is categorically distinct from the execution substrate (Means). MxM decides *what* to do; Means handles *how* it gets done. This separation is architectural, not merely organisational: the same MxM specification drives different Means implementations (local, cloud, air-gapped) without modification.

**HC-3. Jurisdictional grounding.** Each module has a single governing question grounded in a philosophical category:

| Module | Question | Ground |
|--------|----------|--------|
| Mission | What should the system accomplish? | Telos + Mereology |
| Mind | How should the system reason? | Ontic + Epistemic |
| Morals | What must the system never do? | Deontic |
| Memory | What does the system retain across turns? | Temporal |

These categories are non-overlapping by construction. When a practitioner is uncertain where a design decision belongs, the grounding resolves the ambiguity.

---

## 2. Protective Belt

The protective belt contains adjustable auxiliary hypotheses. These are the programme's current best answers to specific architectural questions. They can be modified, replaced, or extended without threatening the hard core.

**PB-1. Channel model.** The current specification names four directional cross-cutting channels:
- Memory to Mind (mid-reasoning recall)
- Mind to Memory (summarisation and persistence decisions)
- Morals to Mission (constraint-driven adjustment)
- Mission to Morals (mission-specific constraint activation)

The number, direction, and semantics of channels are belt hypotheses. Empirical work may identify additional channels (e.g., Memory to Mission for context-driven mission adjustment) or consolidate existing ones.

**PB-2. Bayesian belief revision.** Mind's formal model for reconciling evidence across inference modes (deductive, inductive, abductive) is Bayesian updating. Implementations may approximate this through calibrated confidence bands, verifier outputs, or source-weighting rules. Alternative formal models (Dempster-Shafer, fuzzy logic, argumentation frameworks) could replace Bayesian updating if they better serve specific deployment contexts.

**PB-3. Deontic enforcement pattern.** Morals is specified as executable constraint enforcement: code-level gates that operate independently of the model's output. The current architecture requires defence in depth (multiple independent enforcement layers). The specific enforcement mechanisms (regex matching, AST analysis, secondary LLM validation) and their composition rules are belt hypotheses.

**PB-4. Three-zone memory model.** Memory's session management uses a pinned/summary/verbatim zone structure with token-aware eviction. The number of zones, eviction policies, and compression strategies are adjustable. Alternative memory architectures (hierarchical, episodic, semantic-only) are permissible belt modifications.

**PB-5. Sub-mission mereology.** Mission decomposes into a static base mission (chief end) and dynamic sub-missions (proper parts ordered toward that end). The composition rules, intent classification mechanisms, and sub-mission activation patterns are belt hypotheses. Alternative decomposition strategies (goal trees, task graphs, planning languages) could replace mereological composition.

---

## 3. Positive Heuristic

The positive heuristic directs research toward problems the programme considers tractable and productive. These are the areas where work is expected to yield progressive belt modifications.

**PH-1. Channel formalisation.** Specify cross-cutting channels using an interface definition language or contract specification. This enables automated verification of channel compliance and supports tooling for channel-aware testing. Target: channels as typed, versioned interfaces with machine-checkable contracts.

**PH-2. Multi-model archetypes.** The current model assumes a single primary LLM. Investigate architectures employing multiple models (small classifier for Mission, large model for Mind, specialised model for Morals validation). Map the interaction patterns these introduce onto the channel model.

**PH-3. Dynamic Morals.** The current specification treats Morals constraints as relatively static (configured at deployment, activated by Mission). Investigate adaptive constraint systems that learn from observed violations and adjust enforcement thresholds. This requires a richer model of Morals module internal state.

**PH-4. Tiered adoption patterns.** Not every harness needs full MxM conformance. Develop "minimal MxM" patterns (e.g., Mission + Morals only, or Mind + Memory only) that provide partial architectural benefit with lower adoption cost. Map the tradeoffs explicitly.

**PH-5. Conformance testing.** Develop a test suite that evaluates whether a given harness implementation conforms to the MxM architecture. The conformance checklist in the reference article is a starting point; the target is executable tests.

**PH-6. Empirical validation.** Design controlled experiments comparing MxM-structured harnesses against unstructured alternatives on metrics including task completion, safety, maintainability, and developer productivity. The programme's claims are currently theoretical; empirical confirmation would make it progressive in the Lakatosian sense.

**PH-7. Harness engineering principles beyond MxM.** The programme is broader than the core theory. Principles such as observability, deterministic-first design, defence in depth, and graceful degradation apply to harness engineering generally. Investigate which principles are MxM-specific and which are domain-general.

---

## 4. Negative Heuristic

The negative heuristic directs research *away* from modifications that would compromise the hard core. These are the moves the programme refuses to make.

**NH-1. Do not collapse modules.** If two modules appear to overlap, clarify their jurisdictional boundaries rather than merging them. The four-module structure is hard core.

**NH-2. Do not merge Mind and Morals.** A persistent temptation is to treat constraint enforcement as a cognitive capability ("the model should know not to do that"). This conflates epistemic and deontic concerns. Mind governs how the system reasons; Morals governs what the system must not do. A system prompt instruction is Mind. A code-level gate is Morals. Both are needed; neither subsumes the other.

**NH-3. Do not let Means swallow the cognitive layer.** Tool orchestration, API management, and infrastructure concerns must not absorb the MxM modules. If a framework treats "tool calling" as the primary abstraction and everything else as configuration, it has inverted the architecture: Means is directing cognition rather than serving it.

**NH-4. Do not reduce jurisdictional questions to implementation details.** "What should the system accomplish?" is a telos question, not a prompt-engineering question. "What must the system never do?" is a deontic question, not a content-filtering question. The philosophical grounding is load-bearing: it resolves ambiguity, prevents module collapse, and gives the architecture its non-overlapping jurisdiction. Treating the grounding as decoration removes the hard core's constraint resolution mechanism.

---

## 5. Progressive vs. Degenerating: How to Tell

A Lakatosian programme is assessed by its trajectory, not its current state.

### Signs of Progress

- **Novel predictions confirmed.** The MxM predicts that separating cognitive architecture from execution substrate enables deployment portability. If an MxM-conformant harness is successfully redeployed across model providers or deployment modes without rewriting the cognitive layer, that is a confirmed novel prediction.
- **New implementations conforming.** Independent implementations that adopt the MxM structure and report architectural benefits (testability, maintainability, safety) confirm the programme's productivity.
- **Belt modifications that generate content.** When a belt hypothesis is adjusted (e.g., adding a fifth channel, replacing Bayesian updating with an alternative), and the adjustment yields new testable predictions or design patterns, the modification is progressive.
- **Anomalies absorbed without ad hoc patches.** When a real-world harness presents a concern that does not obviously fit the taxonomy, and the resolution clarifies a module boundary rather than adding a special case, the programme is handling anomalies well.

### Signs of Degeneration

- **Ad hoc patches to save the core.** If every new concern requires a "special exception" or "hybrid module" that blurs the four-module boundary, the taxonomy may not be genuinely exhaustive.
- **Unfalsifiable retreat.** If the programme responds to every counterexample by redefining terms ("that concern is really a sub-concern of Mind"), it is protecting the core through linguistic manoeuvring rather than genuine theoretical work.
- **No novel predictions.** If the programme produces only post hoc rationalisations of existing designs without generating testable claims about new ones, it has stopped being productive.
- **Implementations that nominally conform but gain no benefit.** If adopting the MxM structure adds organisational overhead without improving testability, safety, or maintainability, the architecture may be imposing structure without content.

---

## References

Lakatos, I. (1978) *The Methodology of Scientific Research Programmes: Philosophical Papers Volume 1*. Edited by J. Worrall and G. Currie. Cambridge: Cambridge University Press.

Longmire, J. (2026) 'The MxM: A Reference Architecture for LLM Harness Engineering'. Available at: [theory/4m-reference-architecture/](theory/4m-reference-architecture/index.md).
