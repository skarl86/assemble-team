# skarl86/assemble-team

Single-plugin marketplace for [`assemble-team`](assemble-team) — a Claude Code plugin that converts a user-provided plan into a safely-spawned execution (agent team, single subagent, or printed prompt).

## Install

```
/plugin marketplace add skarl86/assemble-team
/plugin install assemble-team@assemble-team-marketplace
```

The `@assemble-team-marketplace` suffix names the marketplace explicitly. If only one marketplace on your machine exposes `assemble-team`, the shorter form `/plugin install assemble-team` also works.

## Layout

```
.
├── .claude-plugin/
│   └── marketplace.json    # marketplace catalog → source: "./assemble-team"
├── assemble-team/          # the plugin (single-plugin repo: no plugins/ wrapper)
│   ├── .claude-plugin/plugin.json
│   ├── README.md           # plugin-level documentation
│   ├── commands/           # /assemble-team, /enrich-plan, /verify-mapping, /execute-work
│   └── skills/             # 4 sub-skills + 3 reference docs
├── docs/                   # design spec + implementation plan
├── LICENSE
└── README.md               # this file
```

## Plugin documentation

See [`assemble-team/README.md`](assemble-team/README.md) for what the plugin does, slash commands, modes (team / single / manual), and requirements.

## License

MIT. See [LICENSE](LICENSE).
