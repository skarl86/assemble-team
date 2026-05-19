---
name: verify-mapping
description: |
  Step 2 of assemble-team. Accept an enriched plan (from enrich-plan or
  hand-authored matching the format), derive team cards using ROLES.md,
  select an execution mode (team / single / manual) via deterministic
  precedence rules, obtain explicit user approval, and emit an execution
  recipe with a stable recipe_id that downstream execute-work consumes.

  Trigger when the user says:
  - "/verify-mapping ..." (slash command)
  - "map this enriched plan to a team" / "이 plan으로 누가 뭐할지 정해줘"
  - The orchestrator invokes this as step 2 after enrich-plan

  Do NOT use this skill to actually spawn agents — that is execute-work.
  This skill only produces a recipe (markdown).
---

# verify-mapping

Step 2 of the assemble-team v0.2.0 chain. Covers L4 Verification + the new mode selection.

## What this skill does

Given an enriched plan (output of `enrich-plan`), produce an **execution recipe markdown** that names the execution mode and the team cards (or single-agent spec, or manual prompt) the next step will act on. Obtain user approval before emitting; skip not allowed.

## Input contract & validation

**Input**: enriched plan markdown (output of `enrich-plan`, or hand-authored matching that format).

Pre-condition checks (fatal if any fails):

| Check | Failure response |
|---|---|
| Frontmatter present | Reject: "Not an enriched plan. Run `/enrich-plan` first." |
| `forced_blocks_resolved: true` | Reject: "Plan has unresolved gaps. Re-run `/enrich-plan`." |
| `# Why` and `# Tasks` sections present and non-empty | Reject with the specific missing section named |

If the input shape looks like an execution recipe (frontmatter contains `mode:` and `recipe_id:`), reject with: "This input is already a recipe; run `/execute-work` instead."

## Parsing rules (input handling)

These three rules apply to the input the sub-skill receives, before any field-level validation:

1. **Code-fence stripping**: when the input is wrapped in a markdown code fence (e.g., ```` ```markdown\n...\n``` ````), strip the outer fence before extracting the YAML frontmatter. This is a common shape when a user pastes an LLM-formatted block. Log the strip in the output's `notes:` list (e.g., "outer code fence stripped").
2. **Duplicate-frontmatter rejection**: two consecutive `---`...`---` blocks at the very top of the input (separated only by blank lines) → fatal: "Multiple frontmatter blocks detected; remove the extra block." A `---` line that appears inside the body section (e.g., as a markdown horizontal rule after non-trivial content) is NOT a frontmatter block and is ignored by this rule.
3. **Raw-input preservation in fatal errors**: every fatal validation error must include the user's raw input verbatim in the message, so the user can edit and retry without re-typing.

## Output contract

```markdown
---
recipe_id: rcp-<YYYYMMDD-HHMMSS>-<8-char-random>
mode: team | single | manual
team_name: team-<short-purpose>-<YYYYMMDD-HHMM>    # mode=team only; null otherwise
worktree: on | off
permissions: skip | ask | accept_edits
default_branch: <discovered branch>                # mode=team and PR-flow eligible
automation:
  pr_create: true | false
  allow_push: true | false
  skip_qa: true | false
guards_version: "1.0.0"
guards_hash: <12-char hex>                         # sha256 over sorted guard names, prefix-12
notes: []                                          # normalization or fallback log
---

# Team cards    (mode=team)
## <teammate-name>
- role: <one of ROLES.md catalog>
- task: <one-paragraph extract from plan>
- owned: <directory / glob list>
- justification: [<source-tag>]
- model: sonnet | haiku
- permission: inherited | read_only | accept_edits
- plan_approval: on | off

# Single-agent spec    (mode=single)
- role: <one role>
- task: <single task>
- owned: <files>
- worktree: on | off

# Manual prompt    (mode=manual)
<the full synthesized prompt text the user will paste elsewhere>
```

### Canonical names & accepted aliases (auto-normalized on input)

| Canonical | Accepted aliases (input only) |
|---|---|
| `pr_create` | `pr-create`, `prCreate` |
| `skip_qa` | `skip-qa`, `skipQa` |
| `allow_push` | `allow-push`, `allowPush` |
| `accept_edits` | `acceptEdits`, `accept-edits` |
| `read_only` | `read-only`, `readonly` |
| `mode: team` | `Team`, `TEAM` (case-insensitive) |

Log each normalization to `notes:` (`<=80-char` string).

## Mapping derivation

Read `ROLES.md` (sibling file) for the role catalog and Plan → Role heuristic table. For each writer scope in the enriched plan, derive a teammate card with:

- `name` — single word, predictable (`frontend`, `backend`, `qa`, ...)
- `role` — one entry from the ROLES.md catalog
- `task` — one paragraph extracted from the plan
- `owned` — directory / glob list
- `justification` — source tag (`[from-user]` / `[from-code: path:line]` / `[guess]`) per decision
- `model` — role default unless plan overrides
- `permission` — role default; e.g. `read_only` for reviewer
- `plan_approval` — ON for code/DB/destructive tasks

Team size 3–5. If two are needed for the same role, suffix with `-a` / `-b`.

### Dialectic rhythm guard

If this skill has made 3 consecutive decisions without user input during mapping (role inference, file ownership, model default, worktree on/off, team size), inject an `AskUserQuestion` to confirm the next decision before continuing. Counter resets on any user input. Counter scope is per-invocation.

## Mode selection — deterministic precedence

Two pre-passes run first, then a 4-rule cascade. The cascade is "first match wins, no fallthrough."

### Pre-pass A — Mutating capability downgrades

(Always run; mutate the recipe; do NOT pick the mode.)

- `gh` CLI missing AND plan requests PR auto-creation (canonical `automation.pr_create: true`; alias `automation: pr-create` accepted on input): set `pr_create: false` in the recipe and log a `notes:` entry "pr_create disabled: gh CLI not found." Continue.

### Pre-pass B — Hard capability blockers for `team`

(Always run; do NOT pick the mode, but restrict what the cascade can pick.)

- `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS` unset → `team` becomes ineligible.
- `teammateMode` ≠ tmux → `team` becomes ineligible.
- When `team` is ineligible AND the cascade would have picked `team`: substitute `single` if mapping fits one writer (after scope-reduction flow if needed), else block with a remediation checklist. **Manual is offered only as explicit user opt-in**, never as a silent substitution.

### Cascade (first match wins)

1. **Explicit user mode override** — if user invocation specified `mode: <x>`, use `<x>`. Pre-pass B's `team` eligibility check still applies; cascade rule 2's safety constraints still apply.
2. **Safety constraints**:
   - 2+ code-modifying writer cards derived → recommend `team`. If user explicitly requested single, run the §"scope-reduction flow" below.
   - Destructive operation signals (DB migration, drop, force-push) → recommend `team` with `worktree: on, permissions: ask`. Destructive constraints persist regardless of mode.
3. **Plan topology**:
   - 2+ writer scopes (different directories, independent work) → `team`
   - 1 writer scope + auxiliary roles requested in plan (reviewer / qa explicit) → `team`
   - 1 writer scope + no auxiliary roles + intent in {Trivial, Refactor} → `single`
   - 1 writer scope + intent in {Build, Mid-sized} + complexity = Simple → `single`
   - 1 writer scope + complexity = Complex → `team` (auxiliary qa/reviewer recommended)
4. **Convenience default** — `team`.

### `mode=single` scope-reduction flow

When derived mapping has 2+ writer cards (or auxiliary reviewer/qa/researcher) but the user explicitly selects `single`:

1. **Enumerate prunable cards**: list every card to be removed (role + scope + task summary).
2. **Disclose coverage loss**: state what is given up ("the qa role's verdict gate, the reviewer's cross-review and verdict-derived PR banner — will not run").
3. **Explicit yes via `AskUserQuestion`** — silent pruning forbidden.
4. **PR-flow follow-up**: if `automation.pr_create: true`, qa is in the prune list. Ask for an explicit yes to set `automation.skip_qa: true` (and inform of the resulting static PR disclosure). If user declines, set `pr_create: false` and log in `notes:`.
5. **Re-validate**: post-prune mapping must have exactly one writer and no auxiliary roles. Otherwise fatal: "single mode cannot satisfy this plan; switch to team or split the plan."

## User approval prompt (required)

After deriving cards + selecting mode, present to the user:

- Ambiguity score breakdown (from input frontmatter)
- Team cards (or single-agent spec, or manual prompt preview) with source tags
- Worktree decision
- Permission summary
- Applied domain guards (referenced by `guards_version` + `guards_hash`)
- Any `automation:` flags
- The selected mode + recommendation rationale (which precedence rule fired)

Then ask: "이대로 spawn 할까요?" / "Spawn this team?" — proceed only on explicit yes / 진행 / go.

## Recipe ID generation

Generate `recipe_id` as `rcp-<YYYYMMDD-HHMMSS>-<8-char-random>` at the moment of emission. Use:

```bash
recipe_id="rcp-$(date +%Y%m%d-%H%M%S)-$(openssl rand -hex 4)"
```

Or substitute `head -c 8 /dev/urandom | xxd -p` for `openssl` if unavailable.

## Guards version & hash

The canonical guard set is the 8 guards in `ROLES.md` § "Common guards", names 1–8: `Boolean defaults`, `Branching`, `CI / deploy files`, `Commit convention`, `No auto-push / no auto-merge`, `Package manager respect`, `Secrets`, `Teammate spawn guard`.

`guards_version` = `"1.0.0"` for v0.2.0.

`guards_hash` = first 12 hex chars of `sha256(<sorted-guard-names-joined-by-newline>)`. The hash always covers the full bundle (all 8 names), even when a specific mode (e.g., single) injects only a subset at execution time.

Compute with:

```bash
guards_hash=$(printf "%s\n" "Boolean defaults" "Branching" "CI / deploy files" "Commit convention" "No auto-push / no auto-merge" "Package manager respect" "Secrets" "Teammate spawn guard" | sort | sha256sum | cut -c1-12)
```

## When NOT to use

- The plan has not been enriched yet (use `enrich-plan` first).
- The recipe already exists and you just want to run it (use `execute-work`).
