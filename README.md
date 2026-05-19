# skarl86/assemble-team

Single-plugin marketplace for [`assemble-team`](plugins/assemble-team) — a Claude Code plugin that converts a user-provided plan into a safely-spawned execution (agent team, single subagent, or printed prompt).

## Install

```
/plugin marketplace add skarl86/assemble-team
/plugin install assemble-team@skarl86-assemble-team
```

The `@skarl86-assemble-team` suffix names the marketplace explicitly so the install is unambiguous. If only one marketplace exposes `assemble-team`, the shorter form `/plugin install assemble-team` also works.

## Layout

```
.
├── .claude-plugin/
│   └── marketplace.json          # marketplace listing → ./plugins/assemble-team
├── plugins/
│   └── assemble-team/            # the actual plugin
│       ├── .claude-plugin/plugin.json
│       ├── README.md             # plugin-level documentation
│       ├── commands/             # /assemble-team, /enrich-plan, /verify-mapping, /execute-work
│       └── skills/               # 4 sub-skills + 3 reference docs
├── docs/                         # design spec + implementation plan
└── LICENSE
```

Plugin-level documentation lives at [`plugins/assemble-team/README.md`](plugins/assemble-team/README.md).

## License

MIT. See [LICENSE](LICENSE).
