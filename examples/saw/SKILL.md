---
name: saw
description: "Parallel agent coordination: Scout analyzes the codebase and produces a plan; Wave agents implement in parallel. Use for multi-package features, parallel refactors, coordinated changes."
agent-references:
  # Read by inject-agent-context (vendor-neutral) and inject_agent_references hook.
  # Each entry: agent-type + inject (path relative to skill dir) + optional when (regex on prompt).
  - agent-type: scout
    inject: references/scout-suitability-gate.md
  - agent-type: scout
    inject: references/scout-implementation-process.md
  - agent-type: scout
    inject: references/scout-program-contracts.md
    when: "--program"
  - agent-type: wave-agent
    inject: references/wave-agent-worktree-isolation.md
  - agent-type: wave-agent
    inject: references/wave-agent-completion-report.md
  - agent-type: wave-agent
    inject: references/wave-agent-build-diagnosis.md
  - agent-type: wave-agent
    inject: references/wave-agent-program-contracts.md
    when: "frozen_contracts_hash|frozen: true"
  - agent-type: critic-agent
    inject: references/critic-agent-verification-checks.md
  - agent-type: critic-agent
    inject: references/critic-agent-completion-format.md
  - agent-type: planner
    inject: references/planner-suitability-gate.md
  - agent-type: planner
    inject: references/planner-implementation-process.md
  - agent-type: planner
    inject: references/planner-example-manifest.md
  - agent-type: integration-agent
    inject: references/integration-connectors-reference.md
  - agent-type: integration-agent
    inject: references/integration-agent-completion-report.md
---

# Scout-and-Wave: Parallel Agent Coordination

This is an example frontmatter showing how SAW declares `agent-references:` for
subagent reference injection across five agent types.

## What this declares

**scout** — 2 always-injected references + 1 conditional (only when `--program` flag present):
- `scout-suitability-gate.md`: the 4-question assessment that determines if a feature is suitable for SAW
- `scout-implementation-process.md`: 17-step IMPL doc production procedure
- `scout-program-contracts.md`: interface contract rules for program-tier planning (conditional)

**wave-agent** — 3 always-injected + 1 conditional (only when frozen contracts present):
- `wave-agent-worktree-isolation.md`: isolation verification protocol, absolute path patterns
- `wave-agent-completion-report.md`: `sawtools set-completion` format with examples
- `wave-agent-build-diagnosis.md`: H7 build failure diagnosis procedure
- `wave-agent-program-contracts.md`: frozen interface contract enforcement (conditional)

**critic-agent** — 2 always-injected (no conditional references):
- `critic-agent-verification-checks.md`: 6-point brief accuracy verification procedure
- `critic-agent-completion-format.md`: structured CriticResult output format

**planner** — 3 always-injected:
- `planner-suitability-gate.md`: 4-question program-tier assessment
- `planner-implementation-process.md`: 10-step PROGRAM manifest production
- `planner-example-manifest.md`: annotated complete example manifest

**integration-agent** — 2 always-injected:
- `integration-connectors-reference.md`: connector patterns and gap-detection reference
- `integration-agent-completion-report.md`: completion report format

## What this achieves

Each agent type prompt in SAW is a slim identity core (~90–208 lines). The heavy
procedural content — suitability gates, step-by-step procedures, completion formats,
worked examples — lives in the reference files and is injected at launch time.

Without `agent-references:`, each type prompt would need to embed all of this content
unconditionally, or rely on the agent to load references on demand (which it may not do,
or may do too late). The wave-agent type prompt alone went from a large self-contained
document to ~151 lines after extraction.

The `when:` conditional entries ensure context efficiency: `scout-program-contracts.md`
only fires when the `--program` flag is present; `wave-agent-program-contracts.md` only
fires when frozen interface contracts are declared in the agent prompt. Agents that don't
need these references pay no context cost.
