[![Blackwell Systems™](https://raw.githubusercontent.com/blackwell-systems/blackwell-docs-theme/main/badge-trademark.svg)](https://github.com/blackwell-systems)
[![Agent Skills](assets/badge-agentskills.svg)](https://agentskills.io)

The [Agent Skills](https://agentskills.io) spec defines a Resources tier for on-demand reference files and recommends splitting large `SKILL.md` content into focused reference files loaded when needed. The `triggers:` field ([agentskills-subcommand-dispatch](https://github.com/blackwell-systems/agentskills-subcommand-dispatch)) solves this at the orchestrator tier: it loads references into the calling model's context when specific subcommands are invoked.

But orchestrator skills frequently launch subagents — specialized agents for analysis, implementation, review, and integration. These subagents have their own context windows, and they face the same tradeoff the spec leaves open: either their type definition carries every procedure they might ever need (bloated on every launch), or they load references on demand (fragile — the agent may not load them, or loads them too late). The spec says nothing about subagents.

This repo fixes that for agent launch time. Skills declare which reference files to inject for which subagent types. A pre-launch hook injects them into the subagent's initial prompt before it takes its first step — no model judgment required.

The hook mechanism is necessarily different from subcommand dispatch because the target is different: `UserPromptSubmit` adds content to the *calling model's* context; `PreToolUse/Agent` with `updatedInput` modifies the *subagent's initial prompt* before launch. Neither can substitute for the other.

**Scope: subagent launch-time injection.** If you know at launch time which subagent type is being launched, this handles it deterministically. References that depend on the subagent's runtime state are out of scope.

> Works today via YAML extensibility — platforms that understand `agent-references:` act on it, others ignore it. The proposal asks the spec to formally recognize `agent-references:` as a top-level field. Reference implementation for Claude Code (`PreToolUse/Agent`).

## How it works

Skills add `agent-references:` to their YAML frontmatter. Each entry maps a subagent type to a reference file, with an optional `when:` condition. When a subagent of that type launches, matching references are injected into its initial prompt.

Two layers, same declarations:

**Layer 1 — Script (any orchestrator).** `scripts/inject-agent-context` ships with the skill. The orchestrator runs it before launching each subagent:

```bash
inject=$(bash ${SKILL_DIR}/scripts/inject-agent-context \
  --type wave-agent --prompt "$agent_prompt")
full_prompt="${inject}${agent_prompt}"
# pass full_prompt to your agent launcher
```

One instruction in `SKILL.md`: "run `scripts/inject-agent-context` before launching each agent and prepend the output." The orchestrator still decides what to pass — model-initiated, not deterministically enforced. But it eliminates manual per-type reference loading from the orchestrator's instructions.

**Layer 2 — Hook (platform-native).** A `PreToolUse/Agent` hook runs the same script before the subagent launches. No orchestrator decision. The reference is in the subagent's context when it starts. Deterministic. Reference implementation ships for Claude Code.

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
- `when`: optional regex applied to the agent prompt — entry only injects when matched
- Multiple entries per type → all injected in declaration order
- Dedup: HTML comment markers (`<!-- injected: references/X.md -->`) prevent double-injection if the orchestrator and hook both run

## Field Format Constraints

The `agent-references:` field uses the same **portable subset** of YAML as `triggers:`, ensuring reliable parsing across implementations without requiring a full YAML library:

- Flat list of `{agent-type, inject}` entries with optional `when`
- Single-line scalar values — no multi-line strings, no block scalars (`|`, `>`)
- No YAML anchors (`&`), aliases (`*`), or tags (`!!`)
- Values may be quoted (`"..."`) or unquoted

The reference parser (`scripts/inject-agent-context`) uses awk to extract entries without external dependencies. Implementations using a full YAML parser handle this subset correctly; lightweight implementations can also conform.

## Installation

### The injection script (any orchestrator)

Copy `scripts/inject-agent-context` into your skill's `scripts/` directory:

```bash
cp scripts/inject-agent-context ~/.agents/skills/my-skill/scripts/
chmod +x ~/.agents/skills/my-skill/scripts/inject-agent-context
```

Add `agent-references:` to your skill's frontmatter and add an instruction to your `SKILL.md` body for platforms without hook support:

```markdown
**Vendor-neutral fallback:** On platforms without PreToolUse/Agent hook support,
run `bash ${SKILL_DIR}/scripts/inject-agent-context --type <agent-type> --prompt "<prompt>"`
before launching each agent and prepend the output to the prompt.
Agent types: analyst, implementer, ...
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

The hook iterates all skill directories (`~/.claude/skills/`, `~/.agents/skills/`) and delegates to each skill's `inject-agent-context`. Adding a new skill requires zero hook changes.

## Redundancy Model

All three layers are active simultaneously:

| Layer | Mechanism | Platform | Enforcement |
|-------|-----------|----------|-------------|
| Hook | `PreToolUse/Agent` + `updatedInput` | Claude Code | Deterministic (pre-launch) |
| Script | `scripts/inject-agent-context` | Any platform with Bash | Orchestrator-initiated |
| Routing table | Explicit injection instructions in SKILL.md | Any platform | Convention-based |

Users get the best available layer. No regression at any level.

## Relationship to agentskills-subcommand-dispatch

These two repos define complementary layers that cover the full injection surface for orchestrator skills:

```
User types: /my-skill program execute

UserPromptSubmit → inject_skill_context
  Reads:    triggers: in SKILL.md frontmatter
  Matches:  ^/my-skill program
  Injects:  references/program-flow.md
  Target:   Orchestrator context (additionalContext)
  Hook:     UserPromptSubmit

      │
      ▼  (orchestrator runs, calls Agent tool)

PreToolUse/Agent → inject_agent_references
  Reads:    agent-references: in SKILL.md frontmatter
  Matches:  subagent_type = analyst
  Injects:  references/analyst-procedure.md
  Target:   Subagent initial prompt (updatedInput)
  Hook:     PreToolUse/Agent
```

Neither can substitute for the other:
- `UserPromptSubmit` fires at user prompt time — it cannot reach a subagent launched later
- `PreToolUse/Agent` fires at agent launch time — it cannot target the orchestrator's earlier context

Together they complete the spec's progressive disclosure model with deterministic enforcement at both tiers.

## Spec Alignment

This project uses existing Agent Skills conventions:
- `scripts/` directory for executable code ([spec](https://agentskills.io/specification#scripts))
- `references/` directory for on-demand content ([spec](https://agentskills.io/specification#references))

**Today:** `agent-references:` is a top-level frontmatter field. The spec does not currently define it, but YAML parsers ignore unknown fields — platforms that understand `agent-references:` act on it; those that don't ignore it with no behavior change. The alternative (`metadata: agent-references:`) is a poor fit: `metadata:` is defined as a map of string key-value pairs, and `agent-references:` requires structured list data with object entries.

**The proposal:** Ask the spec to formally recognize `agent-references:` as a top-level field intended for platform consumption before agent launch — distinct from `triggers:` (which fires at user prompt time) and distinct from `metadata:` (which is passive author data, not operational platform data). See [`docs/proposal-draft.md`](docs/proposal-draft.md) for the full proposal text.

## Progressive Disclosure Model

The Agent Skills spec defines three progressive disclosure tiers. `triggers:` (agentskills-subcommand-dispatch) provides deterministic loading at the Resources tier for orchestrators. `agent-references:` extends deterministic loading to the subagent tier:

| Layer | Spec name | What | When loaded |
|-------|-----------|------|-------------|
| Discovery | *(extension)* | Skill index in project config | Session start |
| 1 | **Metadata** | `name` + `description` | Session start (catalog) |
| 2 | **Instructions** | Full `SKILL.md` body | Skill activation |
| 3 | **Resources** (orchestrator) | Reference files via `triggers:` | Subcommand dispatch |
| 3 | **Resources** (subagent) | Reference files via `agent-references:` | **Agent launch** (this project) |

## Example

See [`examples/saw/`](examples/saw/) for Scout-and-Wave's complete `agent-references:` configuration: 5 agent types, 14 reference entries, 2 conditional injections. The pattern reduced SAW's agent type prompts by 40–60% while ensuring procedural content is always present in subagent context rather than relying on model recall.

## License

MIT
