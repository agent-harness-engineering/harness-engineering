# Claude Code vs Maestro Agent: Gap Analysis

This document catalogs the harness capabilities observed in Claude Code and assesses what a model-agnostic Maestro Agent must build, adapt, or replace to achieve equivalent governance across arbitrary LLM backends.

## Methodology

Each capability was identified through empirical observation of Claude Code's runtime behavior: tool interception patterns, permission prompts, context management, output formatting, and agent isolation boundaries. Capabilities were then classified by whether they transfer directly to a model-agnostic context, require adaptation, or must be built from scratch.

## Gap Analysis Table

| # | Capability | Claude Code Status | Maestro Agent Requirement | Priority | Gap Type |
|---|-----------|-------------------|-------------------------|----------|----------|
| 1 | Tool Interception Proxy | Built-in: Anthropic tool_use blocks parsed natively | Must intercept and normalize across formats: OpenAI function calling, Anthropic tool_use, raw text action tags | **P0** | Build |
| 2 | Permission Confirmation Channel | Built-in: CLI allow/deny prompts with auto-allow rules | Must support multiple channels: web UI, Telegram, CLI relay. Fail-closed with configurable timeout | **P0** | Build |
| 3 | Self-Exemption Prevention | Partial: config files readable by agent process | Gate config must be read-only to agent process. Filesystem permissions or container boundary required | **P0** | Build |
| 4 | Context Window Management | Built-in: automatic compaction for Claude models | Per-model token counting (tiktoken for GPT, sentencepiece for Gemini, etc.). Pre-compaction triggers. Sliding window: system prompt + persistent memory + recent turns | **P1** | Build |
| 5 | Structured Tool Output Parsing | Built-in: expects Anthropic tool_result format | Parser per model format. Regex fallback for raw text. Schema validation before execution | **P1** | Build |
| 6 | Disposition Output Validation | Minimal: relies on model compliance | Post-generation checks: confidence tags, citation format, structured output schemas. Retry loop on validation failure | **P1** | Build |
| 7 | Agent Isolation | Partial: process-level separation | Filesystem sandboxing, process isolation, resource limits (CPU, memory, network). Container boundary preferred | **P2** | Build |
| 8 | Model-Agnostic Streaming | N/A: Claude-only SSE | Unified streaming adapter normalising SSE across OpenAI, Anthropic, Google, and custom endpoints | **P2** | Build |
| 9 | Skill/Plugin System | Built-in: slash commands, hooks, skill registry | Plugin registry with metadata (name, trigger conditions, model-specific formatting). Dynamic prompt injection. Model-aware formatting | **P3** | Build |
| 10 | Action Tag Processing | Built-in: regex-parsed server-side | Already model-agnostic by design. Regex-parsed server-side, no model-specific dependency | **Done** | Transfer |

## Detailed Gap Descriptions

### 1. Tool Interception Proxy (P0)

**What Claude Code does**: Claude produces `tool_use` content blocks in a known JSON schema. The harness parses these directly and routes to tool handlers.

**What the Maestro Agent needs**: A proxy layer that sits between the model's raw output and tool execution. It must:
- Parse OpenAI-style `function_call` / `tool_calls` JSON
- Parse Anthropic-style `tool_use` content blocks
- Parse raw text containing action tags (e.g., `<tool_call>...</tool_call>`)
- Normalize all formats into a canonical internal representation
- Validate required parameters before forwarding to execution
- Return results in the format the originating model expects

**Key risk**: Misparse leads to wrong tool execution or parameter injection. Schema validation is safety-critical.

### 2. Permission Confirmation Channel (P0)

**What Claude Code does**: CLI prompts the user for allow/deny on each tool call not covered by auto-allow rules. Blocking call; execution halts until response.

**What the Maestro Agent needs**: Multiple relay channels since the orchestrator may run headless (server, CI/CD, background agent). Options:
- **CLI relay**: Direct terminal prompt (local development)
- **Web UI**: Browser-based approval dashboard with session context
- **Telegram/Slack**: Bot-mediated approval for remote operation
- **API webhook**: Programmatic approval for CI/CD integration

All channels must be **fail-closed**: if no response within the configured timeout (default: 5 minutes), the action is denied. The permission decision must be logged with timestamp, channel, approver identity, and action context.

### 3. Self-Exemption Prevention (P0)

**What Claude Code does**: The agent can read its own configuration files, creating a theoretical path where a sufficiently capable model could reason about modifying its own constraints.

**What the Maestro Agent needs**: Hard isolation between the agent process and governance configuration:
- Gate scripts and policy files mounted read-only (filesystem permissions or bind mount)
- Configuration changes require out-of-band human action (separate process, separate credentials)
- Container boundary preferred: agent runs in a container with no write access to the host's governance layer
- Audit log for any attempt to access or modify governance configuration

### 4. Context Window Management (P1)

**What Claude Code does**: Automatic context compaction tuned to Claude's token limits. The system transparently compresses older conversation turns while preserving system prompt and recent context.

**What the Maestro Agent needs**: Model-aware context engineering:
- **Token counting**: Per-model tokenizer (tiktoken for GPT-4/4o, sentencepiece for Gemini, Anthropic's tokenizer for Claude, model-specific for open-weight)
- **Budget allocation**: Three-zone model per 4M Memory spec:
  - Pinned zone: system prompt + tool definitions + persistent memory block (never evicted)
  - Summary zone: running summary of evicted turns (regenerated on each eviction cycle)
  - Verbatim zone: recent messages (oldest evicted when budget exceeded)
- **Pre-compaction triggers**: Initiate compaction at 80% of model's context limit, not at overflow
- **Observation masking**: Replace tool result content in older messages with truncation markers

### 5. Structured Tool Output Parsing (P1)

**What Claude Code does**: Expects tool results in Anthropic's `tool_result` content block format.

**What the Maestro Agent needs**: A parser registry mapping model format to internal canonical form:
- OpenAI: `tool_calls[].function.name` + `tool_calls[].function.arguments` (JSON string)
- Anthropic: `content[].type == "tool_use"` with `name`, `input` fields
- Raw text: regex extraction from `<tool_call>` tags or similar markup
- Validation layer: JSON schema check on extracted parameters before execution
- Error normalization: consistent error format regardless of source model

### 6. Disposition Output Validation (P1)

**What Claude Code does**: Minimal post-generation validation; relies primarily on Claude's instruction-following capability.

**What the Maestro Agent needs**: Post-generation validation pipeline:
- **Confidence tags**: Check for required confidence markers when the mission spec demands them
- **Citation format**: Validate source attribution format against mission-specific schema
- **Structured output**: JSON schema validation when the response must conform to a schema
- **Content policy**: Pattern matching for prohibited content categories
- **Retry loop**: On validation failure, re-prompt with the specific validation error (max 3 retries before escalating to human)

### 7. Agent Isolation (P2)

**What Claude Code does**: Process-level separation between the Claude Code CLI and the model inference. Limited filesystem sandboxing.

**What the Maestro Agent needs**: Defence-in-depth isolation:
- **Filesystem sandbox**: Allowlist of directories the agent may access. All paths validated before I/O.
- **Process isolation**: Agent runs as a separate OS user or in a container with restricted capabilities
- **Resource limits**: CPU, memory, and network quotas (cgroups or container limits)
- **Network policy**: Egress restricted to approved endpoints (LLM API, approved tool endpoints)
- **Temporal limits**: Maximum execution time per tool call and per session

### 8. Model-Agnostic Streaming (P2)

**What Claude Code does**: Streams Claude responses via Anthropic's SSE format with content block deltas.

**What the Maestro Agent needs**: Unified streaming adapter:
- **OpenAI SSE**: `data: {"choices": [{"delta": {"content": "..."}}]}` format
- **Anthropic SSE**: `event: content_block_delta` with `delta.text` format
- **Google SSE**: Gemini streaming response format
- **Generic**: Newline-delimited JSON fallback for custom endpoints
- **Normalization**: All formats emit a common internal event stream that the UI layer consumes
- **Tool call streaming**: Detect partial tool call JSON in stream and buffer until complete before parsing

### 9. Skill/Plugin System (P3)

**What Claude Code does**: Slash commands, hooks (pre/post tool execution), skill registry with metadata. Skills are expanded into full prompts with model-specific formatting.

**What the Maestro Agent needs**: Extensible plugin architecture:
- **Registry**: Plugin manifest with name, description, trigger conditions, required tools, and model compatibility
- **Dynamic prompt injection**: Plugin activation injects its prompt fragment into the active context (Mission sub-mission pattern)
- **Model-specific formatting**: Each plugin may provide format variants for different model families
- **Hook system**: Pre-execution and post-execution hooks for tool calls, with plugin-defined logic
- **Isolation**: Plugins run in a sandboxed context; a faulty plugin cannot crash the orchestrator

### 10. Action Tag Processing (Done)

**What Claude Code does**: Action tags (e.g., `<read_file>`, `<write_file>`) are regex-parsed server-side and routed to tool handlers.

**What the Maestro Agent already has**: This capability is inherently model-agnostic. The regex parser operates on raw text output regardless of which model produced it. No adaptation required -- this transfers directly.

## Summary by Gap Type

| Gap Type | Count | Components |
|----------|-------|------------|
| **Build** | 8 | Tool Proxy, Permission Channel, Self-Exemption Prevention, Context Management, Output Parsing, Disposition Validation, Agent Isolation, Streaming Adapter |
| **Build (deferred)** | 1 | Skill/Plugin System |
| **Transfer** | 1 | Action Tag Processing |

## 4M Module Mapping

Each gap maps to one or more 4M modules, clarifying which governance concern it serves:

| Gap | Primary 4M Module | Secondary |
|-----|-------------------|-----------|
| Tool Interception Proxy | Means | Morals (validation) |
| Permission Confirmation Channel | Morals | -- |
| Self-Exemption Prevention | Morals | -- |
| Context Window Management | Memory | Mind (reasoning quality) |
| Structured Tool Output Parsing | Means | Mind (tool-use planning) |
| Disposition Output Validation | Morals | Mind (metacognition) |
| Agent Isolation | Morals | Means (execution boundary) |
| Model-Agnostic Streaming | Means | -- |
| Skill/Plugin System | Mission | Mind (sub-mission activation) |
| Action Tag Processing | Means | -- |
