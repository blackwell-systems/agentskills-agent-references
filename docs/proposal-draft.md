# Proposal Draft - agentskills/agentskills Issue

**Title:** `Proposal: agent-references: frontmatter field for subagent reference injection`

---

## Background

The spec recommends keeping `SKILL.md` under 500 lines by moving detailed content to `references/`. The `triggers:` proposal ([agentskills-subcommand-dispatch](https://github.com/blackwell-systems/agentskills-subcommand-dispatch)) handles loading those references deterministically into the calling model's context at dispatch time.

This proposal covers the next tier down: the subagents that orchestrator skills launch.

## The gap

Orchestrator skills frequently launch typed subagents: an analyst to plan, an implementer to build, a reviewer to verify. Each subagent has its own context window. Their type definitions face the same tradeoff the spec leaves open for skill bodies: either carry every procedure unconditionally (bloated on every launch), or let the agent load references on demand (fragile; the agent may skip them or read them too late).

The gap is that `triggers:` can't reach here. It fires when the user submits a prompt, adds content to the calling model's context via `additionalContext`, and is long finished by the time an orchestrator is launching subagents mid-execution. Injecting into a subagent's initial prompt requires `PreToolUse` on the `Agent` tool with `updatedInput` to modify the `prompt` parameter before launch. These are different hooks, different events, and different injection targets.

## Proposal

Add `agent-references:` as a recognized top-level frontmatter field:

```yaml
---
name: my-orchestrator
description: Coordinates specialist agents.
agent-references:
  - agent-type: analyst
    inject: references/analyst-procedure.md
  - agent-type: implementer
    inject: references/implementer-protocol.md
  - agent-type: implementer
    inject: references/implementer-program-mode.md
    when: "frozen_contracts_hash|frozen: true"
---
```

Field semantics:
- `agent-type`: matches the `subagent_type` parameter in the platform's agent launch call
- `inject`: path to a Resources tier file, relative to the skill root
- `when`: optional regex tested against the agent prompt for conditional injection
- Multiple entries per type are all injected in declaration order
- HTML comment markers (`<!-- injected: references/X.md -->`) prevent double-injection

Platform behavior: when a `PreToolUse/Agent` hook fires, the platform reads `agent-references:`, finds entries matching the `subagent_type`, tests any `when:` patterns against the agent prompt, and prepends matching reference file contents to the agent's `prompt` parameter via `updatedInput` before launch.

## Why a new top-level field

`triggers:` matches against the user's prompt at invocation time. `agent-references:` matches against a subagent type at launch time and optionally the agent's prompt. They fire at different stages of execution and modify different contexts. Extending `triggers:` semantics to cover both would conflate them.

`metadata:` is the wrong place too. The spec defines `metadata:` as a map of string key-value pairs for passive author data. `agent-references:` is a list of objects consumed by the platform hook before a subagent launches. It's operational data, not metadata, and the string-values constraint rules out the required structure.

## Format constraint

Same portable subset as `triggers:`:

- Flat list of `{agent-type, inject}` entries with optional `when`
- Single-line scalar values; no multi-line strings, no block scalars
- No YAML anchors, aliases, or tags
- Values may be quoted or unquoted

The reference parser uses awk and has no external dependencies, so any implementation can conform without a full YAML library.

## Relationship to triggers:

| Field | Hook event | Target | Injection mechanism |
|-------|-----------|--------|-------------------|
| `triggers:` | `UserPromptSubmit` | Orchestrator context | `additionalContext` |
| `agent-references:` | `PreToolUse/Agent` | Subagent initial prompt | `updatedInput` |

Both declarations live in the same `SKILL.md` frontmatter block. Each field is handled by its own hook at its own stage of execution.

## Backward compatibility

Platforms that don't implement `agent-references:` ignore the field. Standard YAML behavior for unknown keys; no existing skills are affected.

## Production use

`agent-references:` is running in production in the Scout-and-Wave skill ([blackwell-systems/scout-and-wave](https://github.com/blackwell-systems/scout-and-wave)) across 5 agent types and 14 reference entries, including 2 conditional injections. The pattern reduced agent type prompt sizes by 40-60% while keeping procedural content reliably present in subagent context.

## Reference implementation

[blackwell-systems/agentskills-agent-references](https://github.com/blackwell-systems/agentskills-agent-references)

- `scripts/inject-agent-context`: portable bash script that reads `agent-references:` from `SKILL.md` frontmatter and outputs matching reference file contents. Works on any platform via orchestrator instruction.
- `hooks/inject_agent_references`: Claude Code `PreToolUse/Agent` hook. Iterates all installed skill directories and delegates to each skill's `inject-agent-context`. Adding a new skill requires no hook changes.
- `examples/saw/`: complete working example from Scout-and-Wave with annotations explaining what each entry achieves.
