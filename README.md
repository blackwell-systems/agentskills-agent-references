[![Blackwell Systems™](https://raw.githubusercontent.com/blackwell-systems/blackwell-docs-theme/main/badge-trademark.svg)](https://github.com/blackwell-systems)
[![Agent Skills](assets/badge-agentskills.svg)](https://agentskills.io)

Orchestrator skills launch subagents: specialized agents for analysis, implementation, review, and integration. Each subagent has its own context window. Their type definitions face a tradeoff: carry every procedure unconditionally (bloated on every launch), or let the agent load references on demand (the agent may skip them, or read them too late). The [Agent Skills](https://agentskills.io) spec has no mechanism for this tier.

`agent-references:` fills that gap. Skills declare which reference files belong to which subagent types in YAML frontmatter. A `PreToolUse/Agent` hook reads the declarations and injects matching files into the subagent's initial prompt before launch — content is present before the subagent takes its first step.

Scope: subagent launch-time injection. `agent-references:` can only act on information available at launch time (the `subagent_type` and the agent prompt). References that depend on what the subagent discovers at runtime are out of scope.

> Works today via YAML extensibility. Platforms that understand `agent-references:` act on it; others ignore it. The proposal asks the spec to formally recognize `agent-references:` as a top-level field. Reference implementation ships for Claude Code (`PreToolUse/Agent`).

## How it works

Skills add `agent-references:` to their YAML frontmatter. Each entry maps a subagent type to a reference file, with an optional `when:` condition. When a subagent of that type launches, matching references are injected into its initial prompt.

Two layers, same declarations:

**Layer 1 -- Script (any orchestrator).** `scripts/inject-agent-context` ships with the skill. The orchestrator runs it before launching each subagent:

```bash
inject=$(bash ${SKILL_DIR}/scripts/inject-agent-context \
  --type wave-agent --prompt "$agent_prompt")
full_prompt="${inject}${agent_prompt}"
# pass full_prompt to your agent launcher
```

One instruction in `SKILL.md`: "run `scripts/inject-agent-context` before launching each agent and prepend the output." Model-initiated; the orchestrator decides when to call it. Eliminates manual per-type reference loading from the orchestrator's instructions.

**Layer 2 -- Hook (platform-native).** A `PreToolUse/Agent` hook runs the same script before the subagent launches. No orchestrator decision required. The reference is in the subagent's context before its first step. Reference implementation ships for Claude Code.

## Field Definition

Skills declare subagent reference mappings in YAML frontmatter using a top-level `agent-references:` field:

```yaml
---
name: my-orchestrator
description: Coordinates specialist agents to implement features.
agent-references:
  - agent-type: analyst
    inject: references/analyst-procedure.md
  - agent-type: implementer
    inject: references/implementer-protocol.md
  - agent-type: implementer
    inject: references/implementer-completion.md
  - agent-type: implementer
    inject: references/implementer-program-mode.md
    when: "frozen_contracts_hash|frozen: true"
---
```

- `agent-type`: matches the `subagent_type` parameter in the platform's agent launch call
- `inject`: path relative to the skill directory
- `when`: optional regex applied to the agent prompt; entry only injects when matched
- Multiple entries per type are all injected in declaration order
- HTML comment markers (`<!-- injected: references/X.md -->`) prevent double-injection if the orchestrator and hook both run

## Field Format Constraints

The `agent-references:` field uses the same portable subset of YAML as `triggers:`, parseable without a full YAML library:

- Flat list of `{agent-type, inject}` entries with optional `when`
- Single-line scalar values; no multi-line strings, no block scalars (`|`, `>`)
- No YAML anchors (`&`), aliases (`*`), or tags (`!!`)
- Values may be quoted (`"..."`) or unquoted

The reference parser (`scripts/inject-agent-context`) uses awk to extract entries without external dependencies. Full YAML parsers handle this subset correctly; lightweight implementations can also conform.

## Installation

### The injection script (any orchestrator)

Copy `scripts/inject-agent-context` into your skill's `scripts/` directory:

```bash
cp scripts/inject-agent-context ~/.agents/skills/my-skill/scripts/
chmod +x ~/.agents/skills/my-skill/scripts/inject-agent-context
```

Add `agent-references:` to your skill's frontmatter and add a fallback instruction to your `SKILL.md` body for platforms without hook support:

```markdown
On platforms without PreToolUse/Agent hook support, run
`bash ${SKILL_DIR}/scripts/inject-agent-context --type <agent-type> --prompt "<prompt>"`
before launching each agent and prepend the output to the prompt.
```

### The platform hook (optional, deterministic enforcement)

```bash
cp hooks/inject_agent_references ~/.local/bin/
chmod +x ~/.local/bin/inject_agent_references
```

Add to `~/.claude/settings.json`:

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Agent",
        "hooks": [{"type": "command", "command": "inject_agent_references"}]
      }
    ]
  }
}
```

The hook iterates all skill directories (`~/.claude/skills/`, `~/.agents/skills/`) and delegates to each skill's `inject-agent-context`. Adding a new skill requires no hook changes.

## Layers

| Layer | Mechanism | Platform | Enforcement |
|-------|-----------|----------|-------------|
| Hook | `PreToolUse/Agent` + `updatedInput` | Claude Code | Deterministic (pre-launch) |
| Script | `scripts/inject-agent-context` | Any platform with Bash | Orchestrator-initiated |
| Routing table | Explicit injection instructions in SKILL.md | Any platform | Convention-based |

## Relationship to agentskills-subcommand-dispatch

`triggers:` and `agent-references:` operate at different stages of a skill's execution:

```
User types: /my-skill program execute

UserPromptSubmit -> inject_skill_context
  Reads:    triggers: in SKILL.md frontmatter
  Matches:  ^/my-skill program
  Injects:  references/program-flow.md
  Target:   Orchestrator context (additionalContext)
  Hook:     UserPromptSubmit

      |
      v  (orchestrator runs, calls Agent tool)

PreToolUse/Agent -> inject_agent_references
  Reads:    agent-references: in SKILL.md frontmatter
  Matches:  subagent_type = analyst
  Injects:  references/analyst-procedure.md
  Target:   Subagent initial prompt (updatedInput)
  Hook:     PreToolUse/Agent
```

`UserPromptSubmit` fires before the orchestrator starts; it has no access to agent launches that happen later. `PreToolUse/Agent` fires at each agent launch; it has no access to the orchestrator's earlier context. Both declarations live in the same frontmatter block, and each field is handled by its own hook.

## Spec Alignment

This project uses existing Agent Skills conventions:
- `scripts/` directory for executable code ([spec](https://agentskills.io/specification#scripts))
- `references/` directory for on-demand content ([spec](https://agentskills.io/specification#references))

`agent-references:` is a top-level frontmatter field. The spec does not currently define it, but YAML parsers ignore unknown fields, so platforms that understand `agent-references:` act on it and those that don't see no behavior change. Nesting it under `metadata:` would be a poor fit: `metadata:` is defined as a map of string key-value pairs, and `agent-references:` requires structured list data with object entries.

The open proposal asks the spec to formally recognize `agent-references:` as a top-level field for platform consumption before agent launch, distinct from `triggers:` (which fires at user prompt time) and from `metadata:` (passive author data). See [`docs/proposal-draft.md`](docs/proposal-draft.md) for the full proposal text.

## Progressive Disclosure Model

The Agent Skills spec defines three progressive disclosure tiers. `triggers:` provides deterministic loading at the Resources tier for orchestrators. `agent-references:` extends that to the subagent tier:

| Layer | Spec name | What | When loaded |
|-------|-----------|------|-------------|
| Discovery | *(extension)* | Skill index in project config | Session start |
| 1 | **Metadata** | `name` + `description` | Session start (catalog) |
| 2 | **Instructions** | Full `SKILL.md` body | Skill activation |
| 3 | **Resources** (orchestrator) | Reference files via `triggers:` | Subcommand dispatch |
| 3 | **Resources** (subagent) | Reference files via `agent-references:` | Agent launch (this project) |

## Example

See [`examples/saw/`](examples/saw/) for Scout-and-Wave's complete `agent-references:` configuration: 5 agent types, 14 reference entries, 2 conditional injections. The pattern reduced SAW's agent type prompts by 40-60% while keeping procedural content reliably present in subagent context.

## License

MIT
