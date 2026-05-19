# assemble-team

A Claude Code plugin that converts a user-provided plan into a safely-spawned execution. Three sub-skills (Enrich → Map → Execute) are callable independently or chained via `/assemble-team`. Universal — works with any monorepo.

## Install

```
/plugin marketplace add skarl86/assemble-team
/plugin install assemble-team
```

## What it does

`assemble-team` runs three sub-skills in sequence before any agent or team is created:

| Step | Sub-skill | Layers | What happens |
|---|---|---|---|
| 1 | `enrich-plan` | L1 Entry + L2 Routing + L3 Enrichment | Load plan (path/URL/inline) → classify intent × complexity → ambiguity scoring + grilling → emit enriched plan |
| 2 | `verify-mapping` | L4 Verification + new mode selection | Derive team cards from `ROLES.md` → pick mode (team / single / manual) via deterministic precedence → user approval → emit recipe with stable `recipe_id` |
| 3 | `execute-work` | L5 Handoff (mode-branched) | Idempotency check via state dir → execute per mode → emit report |

## Slash commands

| Command | Use |
|---|---|
| `/assemble-team <plan>` | Full chain (orchestrator) |
| `/enrich-plan <plan>` | Step 1 only |
| `/verify-mapping <enriched plan>` | Step 2 only |
| `/execute-work <recipe>` | Step 3 only |

## Modes

| Mode | When | Behavior |
|---|---|---|
| `team` | Default; multi-writer / auxiliary-role plans | `TeamCreate` + monitoring + PR-flow |
| `single` | One writer, no auxiliary roles | One `Agent` call (`subagent_type: general-purpose`) |
| `manual` | User wants to run elsewhere | Print spawn prompts; no actual execution |

## When to use

- The user provides a plan with multiple parallelizable tasks across different scopes.
- The plan covers 3+ teammates' worth of work.
- The user explicitly asks to spawn an agent team.

## When NOT to use

- Single-task work — use a single subagent.
- Strongly sequential work — no parallel benefit.
- Multiple workers would edit the same file — conflict risk (mode=team handles this with worktree=on; mode=single is fine for one writer).

## Files

- `commands/*.md` — slash command shims (4 entry points)
- `skills/assemble-team/SKILL.md` — orchestrator (thin, sequencing + validation)
- `skills/enrich-plan/SKILL.md` — L1+L2+L3, plus `GRILL_PLAN.md` and `PLAN_TEMPLATE.md`
- `skills/verify-mapping/SKILL.md` — L4 + mode selection, plus `ROLES.md`
- `skills/execute-work/SKILL.md` — L5 with team/single/manual branches + idempotency contract

## Requirements

- Claude Code v2.1.32+ with agent-teams capability enabled (`CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1`) for mode=team
- `teammateMode: "tmux"` in `~/.claude/settings.json` for mode=team split-pane teammates
- `gh` CLI (for the optional `automation: pr-create` flow)

## License

MIT. See repository root.
