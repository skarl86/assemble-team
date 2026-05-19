# assemble-team

A Claude Code plugin that converts a user-provided plan into a safely-spawned execution — an agent team, a single subagent, or a printed prompt. The plugin runs a 5-layer harness (Enrich → Map → Execute) as four independently-callable skills, chained by the `/assemble` command.

## Install

```
/plugin marketplace add skarl86/assemble-team
/plugin install assemble-team@assemble-team-marketplace
```

The `@assemble-team-marketplace` suffix names the marketplace explicitly. If only one marketplace on your machine exposes `assemble-team`, the shorter form `/plugin install assemble-team` also works.

## What it does

`assemble-team` runs three sub-skills in sequence, with an orchestrator skill on top:

| Step | Skill | Layers | What happens |
|---|---|---|---|
| 1 | `enrich-plan` | L1 Entry + L2 Routing + L3 Enrichment | Load plan (path / URL / inline) → classify intent × complexity → ambiguity scoring + grilling → emit enriched plan |
| 2 | `verify-mapping` | L4 Verification + mode selection | Derive team cards from `ROLES.md` → pick mode (team / single / manual) via deterministic precedence → user approval → emit recipe with stable `recipe_id` |
| 3 | `execute-work` | L5 Handoff (mode-branched) | Idempotency check via state dir → execute per mode → emit report |
| ★ | `assemble` (orchestrator) | sequencing + handoff validation + halt-resume | Chains the three sub-skills end-to-end |

## Slash commands

| Command | Skill invoked | Use |
|---|---|---|
| `/assemble <plan>` | `assemble` | Full chain (orchestrator) |
| `/enrich-plan <plan>` | `enrich-plan` | Step 1 only |
| `/verify-mapping <enriched plan>` | `verify-mapping` | Step 2 only |
| `/execute-work <recipe>` | `execute-work` | Step 3 only |

## Modes

| Mode | When | Behavior |
|---|---|---|
| `team` | Default; multi-writer / auxiliary-role plans | `TeamCreate` + monitoring + PR-flow |
| `single` | One writer, no auxiliary roles | One `Agent` call (`subagent_type: general-purpose`) |
| `manual` | User wants to run elsewhere | Print spawn prompts; no actual execution |

## Layout

```
.
├── .claude-plugin/
│   ├── marketplace.json    # marketplace catalog (source: "./")
│   └── plugin.json         # plugin manifest
├── commands/               # 4 slash command shims (file name == command name)
│   ├── assemble.md         # → /assemble (orchestrator entry point)
│   ├── enrich-plan.md      # → /enrich-plan
│   ├── verify-mapping.md   # → /verify-mapping
│   └── execute-work.md     # → /execute-work
├── skills/                 # 4 skills (1 orchestrator + 3 sub-skills)
│   ├── assemble/           # orchestrator (chains the 3 sub-skills)
│   ├── enrich-plan/        # + GRILL_PLAN.md + PLAN_TEMPLATE.md
│   ├── verify-mapping/     # + ROLES.md
│   └── execute-work/
├── docs/                   # design spec + implementation plan (historical)
├── LICENSE
└── README.md
```

## When to use

- The user provides a plan with multiple parallelizable tasks across different scopes.
- The plan covers 3+ teammates' worth of work.
- The user explicitly asks to spawn an agent team.

## When NOT to use

- Single-task work — use a single subagent.
- Strongly sequential work — no parallel benefit.
- Multiple workers would edit the same file — conflict risk (mode=team handles this with worktree=on; mode=single is fine for one writer).

## Requirements

- Claude Code v2.1.32+ with agent-teams capability enabled (`CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1`) for mode=team
- `teammateMode: "tmux"` in `~/.claude/settings.json` for mode=team split-pane teammates
- `gh` CLI (for the optional `automation: pr_create` flow)

## Design documentation

- [`docs/superpowers/specs/2026-05-19-assemble-team-v2-design.md`](docs/superpowers/specs/2026-05-19-assemble-team-v2-design.md) — 6-round Codex-reviewed design spec
- [`docs/superpowers/plans/2026-05-19-assemble-team-v0.2.0.md`](docs/superpowers/plans/2026-05-19-assemble-team-v0.2.0.md) — bite-sized implementation plan

## License

MIT. See [LICENSE](LICENSE).
