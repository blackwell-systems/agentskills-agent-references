# Proposal Draft - agentskills/agentskills Issue

**Title:** `Proposal: agent-references: frontmatter field for deterministic subagent reference injection`

---

## Problem

The spec defines a Resources tier for on-demand reference files and recommends
keeping SKILL.md under 500 lines by moving detailed content to references/.
The `triggers:` proposal (see agentskills-subcommand-dispatch) solves deterministic
loading for the orchestrator's context at dispatch time.

But orchestrator skills frequently launch subagents — specialized agents for
analysis, implementation, review, and integration. These subagents have their own
context windows. Their type definitions face the same tradeoff as skill bodies:

- Include all reference content unconditionally: the type prompt is bloated on every
  launch, regardless of whether the agent needs that content for the current task.
- Rely on the agent to load selectively: the agent may not load the file, may load it
  too late, or may not know it exists.

There is no mechanism for skill authors to declare deterministically which reference
files a subagent needs at launch time.

## The mechanism gap

`triggers:` (UserPromptSubmit) cannot solve this. It fires at user prompt time —
before the orchestrator runs. By the time the orchestrator launches a subagent,
UserPromptSubmit is long past and has no access to subagent context.

The correct hook is `PreToolUse` on the Agent tool, with `updatedInput` to modify
the subagent's `prompt` parameter before launch. `additionalContext` (which adds to
the *calling model's* context) is wrong — the target is the subagent's initial
context, not the orchestrator's. This distinction is non-obvious and is the reason
this problem requires its own field rather than reusing `triggers:` semantics.

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

**Field semantics:**
- `agent-type`: matches the `subagent_type` parameter in the platform's agent launch call
- `inject`: path to a Resources tier file, relative to the skill root
- `when`: optional regex tested against the agent's prompt for conditional injection
- Multiple entries per type → all inject (in declaration order)
- Dedup markers (`<!-- injected: references/X.md -->`) prevent double-injection

**Platform behavior:** When a `PreToolUse/Agent` hook fires, the platform reads the
skill's `agent-references:` field, finds entries matching the `subagent_type`, tests
optional `when:` patterns against the agent prompt, and prepends matching reference
file contents to the agent's initial prompt via `updatedInput` before launch.

## Why top-level, not under metadata:

`metadata:` is defined as a map of string key-value pairs — passive author-defined
data. `agent-references:` is structurally different: it is a list of objects consumed
by the platform hook before the subagent launches, not by the model. It is operational
data, not metadata. The same argument applies as for `triggers:`.

## Why a new field, not extending triggers:

`triggers:` matches against the *user's prompt* at invocation time. `agent-references:`
matches against the *subagent type* at launch time and optionally the *agent's prompt*.
They fire at different times, target different contexts, and require different hook
mechanisms. Extending `triggers:` semantics to cover subagents would conflate two
distinct injection surfaces.

## Portable format constraint

Same constraint as `triggers:` — portable subset of YAML parseable without a full
YAML library:

- Flat list of `{agent-type, inject}` entries with optional `when`
- Single-line scalar values — no multi-line strings, no block scalars
- No YAML anchors, aliases, or tags
- Values may be quoted or unquoted

## Relationship to triggers:

Together, `triggers:` and `agent-references:` cover the full injection surface:

| Field | Hook event | Target | Injection mechanism |
|-------|-----------|--------|-------------------|
| `triggers:` | `UserPromptSubmit` | Orchestrator context | `additionalContext` |
| `agent-references:` | `PreToolUse/Agent` | Subagent initial prompt | `updatedInput` |

Neither field can substitute for the other. Both are needed for orchestrator skills
that use progressive disclosure at both tiers.

## Backward compatibility

Platforms that do not implement `agent-references:` ignore the field — standard YAML
behavior for unknown keys. No existing skills are affected.

## Production evidence

`agent-references:` is in production use in the Scout-and-Wave skill
(https://github.com/blackwell-systems/scout-and-wave) across 5 agent types and 14
reference entries. The pattern reduced agent type prompts by 40–60% while improving
output quality (procedural content is always present, not reliant on model recall).

## Reference implementation

https://github.com/blackwell-systems/agentskills-agent-references

Includes:
- `scripts/inject-agent-context` — portable bash script, reads `agent-references:`
  from SKILL.md frontmatter and injects matching references. Works on any platform
  via orchestrator instruction.
- `hooks/inject_agent_references` — Claude Code `PreToolUse/Agent` hook. Iterates
  all installed skills, delegates to each skill's inject-agent-context. Zero changes
  required when adding new skills.
- `examples/saw/` — complete example from the Scout-and-Wave skill (5 agent types,
  14 entries, 2 conditional injections).
