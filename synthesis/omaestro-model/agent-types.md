# omaestro Agent Types

## Design Principle

Each agent type is a single YAML declaration that produces four enforcement configurations.
Types are composable: a type can extend another type and override specific fields.

## Built-In Agent Types

### coder

The primary interactive agent. Full tool access, user-approved execution.

```yaml
agent-type: coder
  governance:
    mind: [confidence-signaling, source-protocol]
    morals: [P1, P2, P3, P4, P5, P6, P7, P8, P9, P10]
    obligations: [O1, O2, O3, O4, O5, O6, O7, O8, O9]
    tools: [bash, edit, write, read, glob, grep, fetch, agent, mcp]
  sandbox:
    filesystem: read-write within project root
    network: allow provider endpoints, deny *
    resources: { cpu: 4, memory: 8G, timeout: none }
  observability:
    trace: true
    cost-budget: none  # user-interactive, no hard limit
  loop:
    provider: user-configured
    permission-mode: auto-edit  # edits OK, bash asks
    context-paths: [CLAUDE.md, project-context.md]
    auto-compact: true
```

### researcher

Read-only exploration agent. Cannot modify files or run arbitrary commands.

```yaml
agent-type: researcher
  governance:
    mind: [source-protocol, confidence-signaling, circularity-detection]
    morals: [P1, P3, P7]
    tools: [read, glob, grep, fetch, ls]
  sandbox:
    filesystem: read-only project root + /tmp/workspace (write)
    network: allow provider endpoints + research APIs, deny *
    resources: { cpu: 1, memory: 2G, timeout: 30m }
  observability:
    trace: true
    eval-dataset: researcher-quality
    cost-budget: $0.50
  loop:
    provider: user-configured (can be cheaper model)
    permission-mode: plan  # read-only, no approval needed
    auto-compact: true
```

### builder

Code generation and modification agent. Can write files but not execute commands.

```yaml
agent-type: builder
  governance:
    mind: [confidence-signaling]
    morals: [P1, P2, P3, P5, P8]
    tools: [read, glob, grep, edit, write, patch, ls]
  sandbox:
    filesystem: read-write within project root (no dotfiles, no .git)
    network: allow provider endpoints, deny *
    resources: { cpu: 2, memory: 4G, timeout: 60m }
  observability:
    trace: true
    cost-budget: $2.00
  loop:
    provider: user-configured
    permission-mode: full-auto  # file ops auto-approved (no bash)
    auto-compact: true
```

### executor

Runs commands but cannot modify source files. For test runners, build systems, CI tasks.

```yaml
agent-type: executor
  governance:
    mind: [confidence-signaling]
    morals: [P1, P2, P4, P6, P9]
    tools: [bash, read, glob, grep, ls]
  sandbox:
    filesystem: read-only project root + /tmp/build (write)
    network: allow provider endpoints + package registries, deny *
    resources: { cpu: 4, memory: 8G, timeout: 60m }
  observability:
    trace: true
    cost-budget: $1.00
  loop:
    provider: user-configured
    permission-mode: auto-edit  # bash commands still require review
    auto-compact: true
```

### coordinator

Manages multi-phase agent workflows. Can spawn agents but not execute tools directly.

```yaml
agent-type: coordinator
  governance:
    mind: [confidence-signaling]
    morals: [P1, P2, P3]
    tools: [agent, read, glob]  # spawn + read-only awareness
  sandbox:
    filesystem: read-only (meta-context write)
    network: allow provider endpoints, deny *
    resources: { cpu: 1, memory: 2G, timeout: 4h }
  observability:
    trace: true
    cost-budget: $5.00  # budget for entire coordinated workflow
  loop:
    provider: user-configured
    permission-mode: plan
    auto-compact: false  # needs full history for coordination
```

## Type Composition

Agent types can extend a base type and override fields:

```yaml
agent-type: security-researcher
  extends: researcher
  governance:
    morals: [P1, P3, P5, P7, P10]  # additional prohibitions
    tools: [read, glob, grep, fetch, ls, bash]  # adds bash for tool inspection
  sandbox:
    network: allow provider + cve.mitre.org + nvd.nist.gov, deny *
    resources: { timeout: 60m }  # longer timeout for deep analysis
```

## Agent Type to Sandbox Policy Mapping

The Type Resolver converts each agent type declaration into a NemoClaw-compatible policy:

| Agent Type | Landlock FS | seccomp | Network | cgroups |
|-----------|------------|---------|---------|---------|
| coder | rw project root | permissive | provider only | generous |
| researcher | ro project + rw /tmp | restrictive | provider + research APIs | tight |
| builder | rw project (no .git) | moderate | provider only | moderate |
| executor | ro project + rw /tmp/build | permissive (needs build tools) | provider + registries | generous |
| coordinator | ro all + rw meta-context | restrictive | provider only | minimal |

## Agent Type to MxM Gate Mapping

The Governance Engine selects gate checks based on agent type:

| Gate | coder | researcher | builder | executor | coordinator |
|------|-------|-----------|---------|----------|-------------|
| Pre-commit hook | Yes | N/A | N/A | Yes | N/A |
| Force-push block | Yes | N/A | N/A | Yes | N/A |
| CDAE (destructive action) | Yes | N/A | Yes | Yes | N/A |
| Content inspection | Yes | N/A | Yes | N/A | N/A |
| Spawn approval | Yes | N/A | N/A | N/A | Yes |
| Branch protection | Yes | N/A | Yes | Yes | N/A |

## Custom Agent Types

Users define custom types in project configuration:

```yaml
# .omaestro/agent-types.yaml
agent-types:
  article-writer:
    extends: builder
    governance:
      mind: [source-protocol, circularity-detection, confidence-signaling]
      tools: [read, glob, grep, write, fetch]
    sandbox:
      filesystem: read-write within content/ directory only
      network: allow provider + research APIs
    observability:
      eval-dataset: article-quality
```
