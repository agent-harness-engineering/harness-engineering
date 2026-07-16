# Implementations

Reference implementations of harness engineering patterns.

The MxM was first extracted from [nemo-harness](https://github.com/jdlongmire/nemo-harness), a self-hosted LLM chatbot built on Nemotron. The nemo-harness implements all four modules and cross-cutting channels in a single-server deployment and serves as the initial reference implementation.

## Adding an Implementation

Implementation studies should document:

1. **Which modules are implemented** and how they map to code
2. **Which channels are active** and their concrete realisation
3. **Means layer**: model provider, deployment mode, tool orchestration
4. **Conformance assessment** against the [MxM checklist](../theory/4m-reference-architecture/index.md#appendix-4m-conformance-checklist)
5. **Lessons learned**: where the MxM structure helped, where it created friction
