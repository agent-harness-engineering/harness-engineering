# Paper 1 — Verifiable Governance for AI Agents: The 4M+1 Framework as Enterprise Architecture

Working draft of paper 1 in the agent-harness-engineering paper series.

## Status

In active drafting. v1 publication target: arXiv preprint + Zenodo DOI.

## Frame

- **Audience:** enterprise architecture practitioners; UAF / MBSE community; AI governance practitioners
- **Form:** UAF-grade architecture description, MBSE discipline, philosophically grounded (reformed-epistemic / telic / deontic / temporal / mereological framing)
- **Contribution:** the 4M+1 framework as a portable, harness-agnostic governance specification with cryptographic verifiability; one reference implementation positioned as the governance plane for a **modular federated digital enterprise** (each capability domain — Dev, Ops, Digital Engineering, ERP, Mod-Sim, … — attaches as an independently-governed module)
- **Naming convention:** the body of the paper speaks abstractly (e.g., "the governance plane", "the security peer agent", "the canonical entry-point file"); concrete reference-implementation names live in [Appendix G](paper-1-appendices-G-H-I.docx) (G.1 term mapping, G.2 repository inventory, G.3 conformance criteria, G.4 module roster)

## Series progression

| # | Audience | Frame |
|---|---|---|
| **1 (this paper)** | Enterprise architecture | UAF / MBSE — framework + reference architecture |
| 2 | AI alignment / philosophy | Reformed-epistemic foundations of 4M+1 |
| 3 | Security research | Cryptographic verification chain; threat-tier model |
| 4 | Operations / case study | Reference deployment lessons |

## Files

- `build_appendices.py` — Python script that generates `paper-1-appendices-G-H-I.docx`. Edit the data dictionaries at the top of the file and rerun to regenerate. Requires `python-docx`.
- `paper-1-appendices-G-H-I.docx` — Working appendices (G: naming map; H: industry benchmarks placeholder; I: internal benchmarks placeholder). Generated; commit alongside the source for review-during-drafting convenience.
- `figures/` — Figure source files (TikZ / SVG / Mermaid sources) and exported figure images for inclusion in the docx draft.

## Outline status

The full section-by-section outline is captured on `agent-harness-engineering/omaestro#2` (synthesis architecture decision tracker) and the work-plan comment on `agent-harness-engineering/omaestro#1`. This subdirectory carries the paper-shaped artifacts as drafting proceeds.

## Build

```bash
python3 build_appendices.py    # regenerates the appendices docx in place
```

## DOI

Reserved at first stable release. Versioned releases via the parent `harness-engineering` repo's tag mechanism; each tagged release mints a fresh Zenodo DOI under the GitHub-Zenodo integration.
