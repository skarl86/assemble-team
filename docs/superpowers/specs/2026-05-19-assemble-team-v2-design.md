# assemble-team v0.2.0 — monorepo migration + structural split

**Date**: 2026-05-19
**Author**: skarl86 (with Claude brainstorming session)
**Status**: design approved, **Codex-reviewed (6 rounds, implementation-ready)**, pending implementation plan
**Target version**: `assemble-team` 0.1.0 → 0.2.0
**Review history**: 6 rounds of `codex:codex-rescue` review against this spec — show-stoppers / significant gaps resolved each round; final verdict "implementation-ready" with no remaining blockers.

## Summary

Migrate the `assemble-team` plugin from the external `skarl86/claude-plugins` install location into this monorepo (`plugins/assemble-team/`), and split its single skill into **three independently callable sub-skills plus one orchestrator**, with a new **mode selection step** (`team` / `single` / `manual`) that gives the user the choice to run without a full agent team.

The 5-layer harness (L1 Entry → L2 Routing → L3 Enrichment → L4 Verification → L5 Handoff) is preserved as the underlying logic. The split rearranges *containers*, not behavior — except for the new mode branching, which is the headline feature.

Memory/preset/project-override systems (Forms A–D discussed during brainstorming) are deferred to a future version. Hooks and agents are not added in this version.

## Goals

1. The plugin lives in this monorepo (`plugins/assemble-team/`), discoverable via the existing `marketplace.json`.
2. The five layers are reorganized into 3 sub-skills + 1 orchestrator, all invokable independently or as a chain.
3. A new mode selection step lets the user choose between team-spawn, single-subagent, or print-only execution.
4. All current v0.1.0 behaviors are preserved when `mode=team` is selected (zero regression on the happy path).

## Non-goals (v0.2.0)

- Memory / presets / `.claude/assemble-team.local.md` project overrides — deferred to v2.
- Auto-memory / pattern detection — deferred.
- Hooks (SessionStart, PreToolUse, etc.) — deferred.
- Sub-agents (Agent file definitions) — deferred; current mode=single uses the runtime `Agent` tool, not a packaged agent definition.
- Multi-language UI beyond Korean/English as they already work.

## Architecture

### Directory layout

```
plugins/assemble-team/
├── .claude-plugin/
│   └── plugin.json                 # version 0.2.0
├── README.md                       # updated for new structure
├── commands/
│   ├── assemble-team.md            # /assemble-team — run all three sub-skills
│   ├── enrich-plan.md              # /enrich-plan — step 1 only
│   ├── verify-mapping.md           # /verify-mapping — step 2 only
│   └── execute-work.md             # /execute-work — step 3 only
└── skills/
    ├── assemble-team/              # thin orchestrator
    │   └── SKILL.md
    ├── enrich-plan/                # L1 + L2 + L3
    │   ├── SKILL.md
    │   ├── GRILL_PLAN.md           # migrated verbatim from v0.1.0
    │   └── PLAN_TEMPLATE.md        # migrated verbatim from v0.1.0
    ├── verify-mapping/             # L4 + mode selection
    │   ├── SKILL.md
    │   └── ROLES.md                # migrated verbatim from v0.1.0
    └── execute-work/               # L5, mode-branched
        └── SKILL.md
```

### Call flow — chained

```
User → /assemble-team "<plan or path>"
        │
        ▼
[assemble-team]  orchestrator: chains the three sub-skills
        │  raw plan
        ▼
[enrich-plan]    L1: load (path/URL/inline)
                 L2: classify intent × complexity
                 L3: ambiguity score + grilling (via GRILL_PLAN.md)
        │  enriched plan (frontmatter + sectioned markdown)
        ▼
[verify-mapping] L4: derive team cards (via ROLES.md) + mode selection + user approval
        │  execution recipe (frontmatter with mode + team_cards / single_agent / manual_prompt)
        ▼
[execute-work]   L5: branch on mode
                 team   → TeamCreate + monitoring + PR-flow (current behavior)
                 single → Agent tool, one subagent, no SendMessage
                 manual → print spawn prompt(s), do not execute
        │  execution report (frontmatter + per-teammate or single summary)
        ▼
       User
```

### Call flow — standalone

Each sub-skill can be invoked independently for partial workflows:

- `/enrich-plan "<raw plan>"` → returns the enriched plan markdown; halts (no further chaining).
- `/verify-mapping "<enriched plan>"` → returns the execution recipe markdown; halts.
- `/execute-work "<execution recipe>"` → executes per the recipe's `mode`; returns the execution report.

The orchestrator (`/assemble-team`) owns sequencing, handoff validation, and halt-resume control flow — but no *mapping* or *execution* business logic (those live in the sub-skills). If any sub-skill fails or the user aborts, the partial output is preserved and the chain halts. See §"Orchestrator: `assemble-team`" below for the validation steps the orchestrator performs at each handoff.

## Components

### Sub-skill: `enrich-plan`

**Responsibility**: Accept any plan-shaped input, classify it, and fill its gaps until it is concrete enough to map.

**Layers covered**: L1 Entry + L2 Routing + L3 Enrichment.

**Input**: one of
- File path (relative or absolute)
- GitHub issue/PR URL (parsed with explicit `--repo <owner>/<repo>`)
- Inline plan body text

**Output**: enriched plan markdown:

```markdown
---
ambiguity_score: 0.24
classification:
  intent: Mid-sized | Trivial | Refactor | Build | Collaborative | Architecture | Research
  complexity: Trivial | Simple | Complex
sources:
  goal: from-user | from-code:<path:line> | guess
  scope: ...
  tasks: ...
  constraints: ...
  dod: ...
forced_blocks_resolved: true | false
guesses_to_confirm: [<list of fields tagged [guess] needing user check>]
notes: []   # list of <=80-char strings logging any normalization or fallback applied at this step
---

# Why
[<source-tag>] <body>

# Scope
[<source-tag>] <paths>

# Tasks
- [<source-tag>] <task line>
- ...

# Constraints
[<source-tag>] <constraints>

# Definition of Done
[<source-tag>] <DoD>

# Out of scope
[<source-tag>] <exclusions>

# Risks
[<source-tag>] <risk lines>
```

**Behavior preserved from v0.1.0**:
- Pre-gate (block short / nounless plans)
- GitHub fetch with explicit `--repo` parsing
- Intent × Complexity classification per ROLES.md signals
- Ambiguity score thresholds (≤0.30 pass; >0.50 force-block-aware grilling)
- Forced-block heuristics (Goal missing / Tasks==0 / Constraints missing + destructive)
- 7-step grilling dependency order via GRILL_PLAN.md
- Source tagging on every filled field
- User-impatience collapse rules
- Spawn discipline: never call TeamCreate from this sub-skill

### Sub-skill: `verify-mapping`

**Responsibility**: From an enriched plan, derive the execution recipe — team cards (or single-agent spec, or manual prompt) — and obtain explicit user approval.

**Layers covered**: L4 Verification + new mode selection.

**Input**: enriched plan markdown (output of `enrich-plan`, or hand-authored matching the format). Pre-condition check: frontmatter must contain `forced_blocks_resolved: true`. If false, reject with guidance to run `enrich-plan` first.

**Output**: execution recipe markdown:

```markdown
---
recipe_id: rcp-<YYYYMMDD-HHMMSS>-<8-char-random>   # stable id for idempotency
mode: team | single | manual
team_name: team-<short-purpose>-<YYYYMMDD-HHMM>    # mode=team only
worktree: on | off
permissions: skip | ask | accept_edits             # snake_case canonical
default_branch: <discovered branch>                # mode=team and PR-flow eligible
automation:
  pr_create: true | false
  allow_push: true | false
  skip_qa: true | false
guards_version: <semver of the bundled guards>     # e.g. "1.0.0"
guards_hash: <12-char hex>                         # see §"Guards hash specification" below
notes: []                                          # normalization or fallback log from this step
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

**Canonical names and accepted aliases** (verify-mapping always emits canonical form; parser accepts aliases on input for tolerance):

| Canonical (frontmatter) | User-facing alias (plan body, dialog) |
|---|---|
| `pr_create` | `pr-create` |
| `skip_qa` | `skip-qa` |
| `allow_push` | `allow-push` |
| `accept_edits` | `acceptEdits`, `accept-edits` |
| `read_only` | `read-only`, `readonly` |

**Mode selection — deterministic precedence**:

Two pre-passes run first, then a 4-rule cascade. The cascade is "first match wins, no fallthrough."

**Pre-pass A — Mutating capability downgrades** (always run; mutate the recipe; do NOT pick the mode):
- `gh` CLI missing AND plan requests PR auto-creation (canonical: `automation.pr_create: true`; alias `automation: pr-create` accepted on input): set `pr_create: false` in the recipe and log a `notes:` entry "pr_create disabled: gh CLI not found." Continue.
- (Future capability downgrades that mutate automation but don't constrain mode go here.)

**Pre-pass B — Hard capability blockers for `team`** (always run; do NOT pick the mode, but restrict what the cascade can pick):
- `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS` unset → `team` becomes ineligible for the cascade.
- `teammateMode` ≠ tmux → `team` becomes ineligible for the cascade.
- When `team` is ineligible AND the topology cascade would have picked `team`: substitute `single` if the mapping fits one writer (after the scope-reduction flow if needed), else block with a remediation checklist. **Manual is offered only as explicit user opt-in**, never as a silent substitution (since manual abandons execution).

**Cascade rules** (first match wins, no fallthrough):

1. **Explicit user mode override** — if the user invocation specified `mode: <x>` (in plan body, earlier dialog, or `AskUserQuestion` answer), use `<x>`. Pre-pass B's `team` eligibility check still applies (if `<x> = team` but `team` ineligible, substitute per pre-pass B). Cascade rule 2's safety constraints still apply (if `<x> = single` but mapping has 2+ writers, run scope-reduction flow).
2. **Safety constraints** (interact with explicit user picks, but cannot be bypassed silently):
   - 2+ code-modifying writer cards derived. The derivation itself is the safety-relevant state, not the mode pick. Three sub-cases:
     - User did not request single → recommend `team` (only safe mode).
     - User explicitly requested single → run the **scope-reduction flow** (see §"`mode=single` scope-reduction flow" below) which prunes cards via explicit user yeses, then re-validates the resulting recipe. The recipe is only emitted when the post-prune mapping has exactly one writer; otherwise the request is fatal.
   - Destructive operation signals (DB migration, drop, force-push) → recommend `team` with `worktree: on, permissions: ask`. User may override mode but the destructive-op constraints (`worktree: on, permissions: ask`) persist regardless of mode choice.
3. **Plan topology** (when no override / no safety force):
   - 2+ writer scopes (different directories, independent work) → `team`
   - 1 writer scope + auxiliary roles requested in plan (reviewer / qa explicit) → `team`
   - 1 writer scope + no auxiliary roles + intent in {Trivial, Refactor} → `single`
   - 1 writer scope + intent in {Build, Mid-sized} + complexity in {Simple} → `single`
   - 1 writer scope + complexity = Complex → `team` (auxiliary qa/reviewer recommended for safety)
4. **Convenience defaults** (when no rule above produces a recommendation):
   - Default = `team` (preserves v0.1.0 baseline behavior).

**`mode=single` scope-reduction flow**:

When the derived mapping has 2+ writer cards (or any auxiliary roles such as reviewer/qa/researcher) but the user explicitly selects `single`, verify-mapping must run this flow before emitting the recipe:

1. **Enumerate prunable cards**: list every card that would be removed to reach a one-writer mapping. Show the user the full list with role + scope + task summary.
2. **Disclose coverage loss**: state what is being given up — "the qa role's verdict gate, the reviewer's cross-review and verdict-derived PR banner, all auxiliary safety nets — will not run."
3. **Explicit yes via `AskUserQuestion`**: the user must affirmatively confirm the prune. Silent pruning is forbidden.
4. **PR-flow follow-up**: if the plan also has `automation.pr_create: true` (or alias `automation: pr-create`), the qa card is in the prune list. Verify-mapping must additionally ask for an explicit yes to set `automation.skip_qa: true` (and inform the user of the resulting static PR disclosure). If the user declines, set `pr_create: false` and log in `notes:`.
5. **Re-validation**: after pruning, the mapping must have exactly one writer card and no auxiliary roles. If the pruned mapping still has 2+ writers (e.g., the plan genuinely needs two writers), the request is fatal: "single mode cannot satisfy this plan; switch to team or split the plan."

The flow exists because the user's explicit `single` pick deserves a hearing, but the underlying safety concern (Git index race for 2+ writers) is not negotiable.

**`mode=manual` when execution was reachable**:

If the user picks `manual` while `team` or `single` was still capability-feasible, verify-mapping confirms once ("execution is possible; manual will only print prompts — proceed?") and emits the recipe. No automation flag is honored under manual.

**Behavior preserved from v0.1.0**:
- Team size 3–5 (mode=team)
- Dialectic rhythm guard (3 consecutive non-user decisions → AskUserQuestion)
- Explicit user approval gate (skip not allowed)
- Source justification on every card decision
- Common guards auto-injected (records `guards_version` and `guards_hash` for verifiable injection; the hash covers the sorted list of guard names that will be inlined into every spawn prompt)

### Sub-skill: `execute-work`

**Responsibility**: Execute the recipe per its `mode` field.

**Layers covered**: L5 Handoff (mode-branched).

**Input**: execution recipe markdown (output of `verify-mapping`, or hand-authored matching the format). Pre-condition: `mode` field present and valid.

**Output**: execution report markdown:

```markdown
---
recipe_id: <stable id copied from the execution recipe>
mode: team | single | manual
team_name: <same value as the recipe's team_name>      # mode=team only; null otherwise
result: completed | failed | aborted | printed | already_started   # printed = manual; already_started = idempotency hit
pr_url: <https URL>                                    # when PR created
duration_seconds: <int>
verdicts:                                              # mode=team only; null for single/manual
  qa: all-pass | partial-pass | failed | null
  reviewer: { errors: N, warnings: N, ok: N } | null
notes: []                                              # list of <=80-char strings
---

# Summary
<one paragraph>

# Per-teammate reports   (mode=team)
## <teammate-name>
- commit: <message> [<sha>]
- report: <one line>

# Single-agent report    (mode=single)
- commit: ...
- report: ...

# Manual prompt output   (mode=manual)
<the prompts that were printed>
```

#### Mode=team (preserved from v0.1.0)

Full L5 behavior:
- `TeamCreate` with `team-<short-purpose>-<YYYYMMDD-HHMM>` naming
- Per-teammate spawn prompt injection (8 components: role body, common guards, plan, task, owned files, worktree path, collaboration rules, DoD)
- Worktree isolation per existing heuristics
- Idle/stuck/all-idle monitoring
- PR-flow gate (canonical `automation.pr_create: true`): qa verdict, reviewer verdict, DEFAULT_BRANCH discovery, push reconciliation, non-interactive `gh pr create --title --body-file`
- `automation.skip_qa` and `automation.allow_push` flag handling (hyphenated aliases `skip-qa` / `allow-push` accepted on input)
- Cleanup on user "cleanup" / "팀 정리"

#### Mode=single (new)

- Exactly one code-modifying agent. No qa/reviewer/researcher cards in this mode.
- Calls the runtime `Agent` tool with `subagent_type: "general-purpose"`. Rationale: general-purpose has the full tool set (Read/Edit/Write/Bash/Glob/Grep) needed by a code-modifying role, and does **not** restrict any tools the single agent might need. The current ROLES.md spawn guard #8 (forbidding restrictive `subagent_type` for roles requiring team-collaboration tools like `SendMessage` / `TaskUpdate`) **does not apply** because single mode is non-collaborative by construction — there is exactly one agent and no SendMessage path. This non-application is stated explicitly so future readers do not file it as a guard violation.
- The Agent invocation prompt is constructed inline (not by referring to an external agent file) and contains, in order:
  1. **A single-mode preamble** that neutralizes collaboration references inside the role body. Exact text:
     ```
     You are running in mode=single. There is no team, no other teammates, no SendMessage, no TaskUpdate signaling, and no inter-agent coordination. Any references to "the lead", "teammates", "SendMessage", "TaskUpdate", or "collaboration" inside the role body below DO NOT apply to you. Where the role body asks you to "report to the lead via SendMessage", interpret this as: include the same content as your final reply text.
     ```
     This preamble exists because `ROLES.md` is migrated verbatim from v0.1.0 and its role bodies (frontend, backend, reviewer, etc.) reference SendMessage and the lead. The preamble redirects those references rather than editing `ROLES.md` (which must remain a verbatim move).
  2. The role body inline-injected from `ROLES.md` (same content that mode=team would inject — verbatim, no edits).
  3. The applicable subset of common guards — guards 1–7 (commit convention, branching, CI/deploy file restriction, package-manager respect, secrets, boolean defaults, no auto-push/merge). Guard 8 (the team-collaboration spawn guard) is omitted as inapplicable; the omission is logged in the execution report's `notes:` field.
  4. The plan body (shared context).
  5. The agent's task paragraph (from the recipe's `# Single-agent spec` block).
  6. Owned files / glob list.
  7. Worktree path + branch when isolation is on.
  8. Definition of Done (single-agent variant: tests pass + Conventional Commits commit + one-paragraph final reply summarizing the change).
- No SendMessage, no inter-agent coordination, no TaskUpdate signaling.
- Worktree: default off (single writer, no race). Honor plan signal if specified.
- PR-flow:
  - **Reviewer-banner injection is unavailable in mode=single.** Single mode has no reviewer role, so no reviewer verdict can be produced and no reviewer-derived banner is added to the PR body. This is a removal of the v0.1.0 capability for this mode, not a degraded version of it.
  - **A static disclosure notice is the only mode=single PR addition.** When `pr_create + skip_qa` are both explicitly confirmed by the user (skip_qa cannot be inferred — see Mode selection §3), execute-work runs the same noninteractive `gh pr create` flow as mode=team, but prepends a fixed disclosure block to `--body-file`:
    ```
    > **assemble-team mode=single**: this PR was produced by a single subagent.
    > No reviewer verdict and no QA verdict were collected.
    > The human PR reviewer is the only review gate.
    ```
    This is a constant string, not a reviewer-summary banner — its source is the plugin itself, not a reviewer role's output.
  - Without `skip_qa`, blocks PR creation and reports the missing flag with a remediation message. No silent retry.
- Output: subagent's final response normalized into the report format with `verdicts: null` (no team verdicts exist).

#### Mode=manual (new)

- `execute-work` synthesizes the spawn prompt(s) and **prints** them. No TeamCreate, no Agent call.
- For multi-card recipes, each teammate's prompt is printed in its own fenced block, labeled by teammate name.
- Automation flags are ignored (manual = user does it themselves).
- Report's `result: printed`.

### Orchestrator: `assemble-team`

**Responsibility**: Sequence the three sub-skills and own the validation / halt-resume control flow between them. No *mapping* or *execution* business logic (those belong to the sub-skills); but the orchestrator does enforce input/output contracts and idempotency at each handoff.

Behavior:
1. Receive initial input (raw plan).
2. Invoke `enrich-plan` skill via the Skill tool.
3. Validate output (parser policy below; required field `forced_blocks_resolved: true`; bail if `guesses_to_confirm` is non-empty — must be resolved before proceeding).
4. Invoke `verify-mapping` skill.
5. Validate output (required: `recipe_id`, `mode`, `guards_version`, `guards_hash`).
6. Invoke `execute-work` skill, passing the recipe.
7. Return the final execution report to the user.

On any sub-skill failure or user abort: halt, present the partial output to the user, note the next sub-skill to resume from. The user may pass the partial output back into a downstream sub-skill manually (`/verify-mapping "<paste enriched plan>"`) to continue.

### Idempotency contract

The orchestrator and `execute-work` cooperate to make the chain safe to re-run after partial failure or accidental duplicate invocation.

#### Identifiers and state files

| Element | Owner | Purpose |
|---|---|---|
| `recipe_id` | verify-mapping (emits) | Stable identifier for a single execution attempt. Format: `rcp-<YYYYMMDD-HHMMSS>-<8-char-random>`. |
| `~/.claude/plugins/assemble-team/state/started/<recipe_id>.md` | execute-work | Sentinel that an execution has begun. Body: timestamp, mode, team_name (mode=team only), plan title, owned scope list. |
| `~/.claude/plugins/assemble-team/state/completed/<recipe_id>.md` | execute-work | Sentinel that an execution finished. Body: result, duration_seconds, pr_url (when applicable), short summary. |
| `~/.claude/plugins/assemble-team/state/aborted/<recipe_id>.md` | execute-work | Sentinel that a started run was abandoned by the user. |

#### Single-location invariant

**At most one of `{started/, completed/, aborted/}` contains `<recipe_id>.md` at any moment.** All state transitions move the sentinel atomically; no transition creates or leaves a second copy. The pre-execution lookup relies on this — if it ever observes two locations populated for the same `recipe_id`, the contract is violated and execute-work treats this as an anomaly.

State transitions:

| Transition | Operation | Atomicity |
|---|---|---|
| Fresh run starts | Atomic claim (see below) creates `started/<recipe_id>.md` | `set -C` (noclobber) on a flat-file sentinel + temp-then-rename — see "Atomic claim mechanism" |
| Successful completion | `mv state/started/<recipe_id>.md state/completed/<recipe_id>.md` | POSIX `mv` within a single filesystem is atomic |
| Terminal failure (not user-aborted) | `mv state/started/<recipe_id>.md state/completed/<recipe_id>.md` (body records the failure) | atomic |
| User abort | `mv state/started/<recipe_id>.md state/aborted/<recipe_id>.md` | atomic |
| User restart | `rm state/started/<recipe_id>.md`, then perform fresh atomic claim | sequence; the brief window between rm and claim is fine — a concurrent caller would claim and the restarter would lose, in which case the restarter follows the resume/abort path on the next pre-execution lookup |
| Reading a `completed/` or `aborted/` sentinel after the fact | read-only | n/a |

Note: there is **no** "restart-after-abort" transition. Once a recipe is aborted, the recipe_id is *retired*. The user re-runs verify-mapping to get a fresh recipe_id.

#### Pre-execution lookup (5-row state table)

The sentinel for any state is **always a flat markdown file**: `started/<recipe_id>.md`, `completed/<recipe_id>.md`, or `aborted/<recipe_id>.md`. Never a directory.

**Lookup predicate**: the lookup tests for the exact path `<dir>/<recipe_id>.md` (no wildcard or glob), e.g. `[ -f "$STATE/started/${recipe_id}.md" ]`. Temp files are never mistaken for sentinels because their names differ by the leading dot (`.${recipe_id}.md.tmp` vs `${recipe_id}.md`) — the exact-path test naturally excludes them with no special filter needed.

Before any side effect, execute-work checks state for the `recipe_id`:

| State | Action |
|---|---|
| absent in all three subdirectories | Fresh run. Perform atomic claim (creates `started/<recipe_id>.md`). On claim failure (race lost between `set -C` test and write), re-enter this lookup — the second time it will see "in `started/` only" and follow that branch. |
| present in `started/` only | Incomplete run. Ask user: **resume** (mode=team only — see resume semantics), **restart** (delete `started/<recipe_id>.md`, perform fresh atomic claim, proceed), or **abort** (mv started→aborted, emit `result: already_started`, exit). On 60s silence default to **abort**. A 0-byte `started/<recipe_id>.md` is treated identically (partial sentinel = same lookup outcome). |
| present in `completed/` only | Previously completed. Emit `result: already_started` with no side effects, no prompting. The execution report references the original `completed/<recipe_id>.md` body. |
| present in `aborted/` only | Recipe_id retired. Block with: "This recipe_id was aborted previously and cannot be reused. Re-run /verify-mapping to obtain a new recipe_id." |
| present in ≥ 2 subdirectories | Anomaly (single-location invariant violated). Fatal: log directory state; ask user to inspect `~/.claude/plugins/assemble-team/state/`. |

#### Atomic claim mechanism (noclobber + temp-rename)

The sentinel is always a flat `.md` file. Claim atomicity is provided by `set -C` (noclobber), which makes shell `>` redirection use `open(O_CREAT|O_EXCL)` — the file is either created (claim succeeded) or the call fails (someone else already claimed). Body write uses temp-then-rename so partial writes never leave inconsistent observed state.

`set -C` is mandated by POSIX (POSIX.1-2017 §2.9.2) and works in Bash 2+ and all POSIX-conformant shells; no minimum Bash version constraint is needed.

Concrete shell sequence (execute-work runs this via the `Bash` tool):

```bash
SENTINEL="$STATE/started/${recipe_id}.md"
# Per-process unique temp path. $$ is the shell PID; including it ensures two
# concurrent invocations of execute-work with the SAME recipe_id never share a
# temp filename, so a losing claimant cannot delete or corrupt the winner's
# in-flight body write. The dot prefix keeps temps invisible to the lookup
# predicate (exact-path test on `<recipe_id>.md`).
TMP="$STATE/started/.${recipe_id}.md.$$.tmp"

# Step 1: atomic claim. set -C makes ">" fail if SENTINEL already exists.
# The subshell ( ... ) scopes noclobber to this command only — do not flatten to
# `set -C; : > "$SENTINEL"` because that would leak noclobber to the outer shell.
if ! ( set -C; : > "$SENTINEL" ) 2>/dev/null; then
  # File already exists — claim failed; follow lookup-present path.
  exit_with_lookup_present
fi

# Step 2: write the body via temp-then-rename so partial writes do not leave
#         a half-written observable sentinel. After step 1 SENTINEL is a 0-byte
#         flat file; step 2 replaces it with the full body via mv (atomic on POSIX).
printf '%s\n' "$body" > "$TMP"
mv "$TMP" "$SENTINEL"
# After successful mv, $TMP no longer exists. If the process dies before mv,
# the leftover $TMP is inert: its filename includes this process's PID, so no
# other process touches or depends on it.
```

There is no defensive temp-cleanup step before the claim. An earlier draft of this spec ran `rm -f "$TMP"` first, but with a *shared* temp filename that step opened a race: a losing claimant could delete the winning claimant's in-flight body file. Per-process temp naming eliminates the need for any pre-claim cleanup.

Crash-safety analysis (per-die-point):

- Process dies before step 1: nothing happened; next caller starts fresh.
- Process dies *during* step 1 (after `:` redirect created the empty file but before subshell exits): SENTINEL exists as 0-byte file. Next lookup sees "started/ only" and asks resume/restart/abort. 0-byte and partially-written sentinels are both treated as "incomplete" — both are acceptable.
- Process dies during step 2's `printf` write to TMP: SENTINEL is still 0-byte; TMP exists partially with this process's PID in the name. The lookup ignores TMP (exact-path test) and the partial TMP is inert (no other process depends on PID-suffixed temps). Lookup outcome: "started/ only" → resume/restart/abort.
- Process dies between TMP write and `mv`: SENTINEL is still 0-byte; TMP has the full body but invisible to the lookup. Same outcome as above.
- Process dies during `mv` itself: `mv` within a single filesystem is atomic at the kernel level — either SENTINEL is still 0-byte (mv hadn't started) or SENTINEL is the full body (mv completed). No middle state.

There is **no orphaned-lock state**: the sentinel is always a flat file in one of three known locations, never a directory and never a separate `.lock` artifact. Temp files (`.${recipe_id}.md.<pid>.tmp`) are sibling files that the lookup explicitly ignores and that never collide between concurrent processes.

**Stale temp accumulation**: temp files from killed processes are not auto-cleaned. They are inert (lookup ignores them) but may accumulate over time. Manual cleanup is `rm "$STATE"/{started,completed,aborted}/.*.tmp` and can be done at any time without risk because each temp file is only depended on by its (now-dead) owner process. A future plugin update may add `assemble-team gc`; documented limitation for v0.2.0.

Cross-machine concurrency is not in scope (recipe_ids are random; collisions are extremely improbable). This mechanism is sufficient for single-machine concurrency.

#### Resume semantics (mode=team only)

When the user picks **resume**:
- Read `team_name` from `started/<recipe_id>.md`.
- Use the `TaskList` tool (which must be in `/execute-work`'s `allowed-tools`) to enumerate live team task status.
- Skip any teammate whose task has reached its DoD (commit + report present).
- Re-spawn or extend any teammate not yet idle.
- For mode=single and mode=manual, resume is unavailable — execute-work answers with "single/manual mode does not support resume; choose restart or abort."

#### Cleanup policy and portability

- **Cleanup**: not auto-cleaned in v0.2.0. Files accumulate; the user may purge `state/` manually. Future versions may add an `assemble-team gc` command if files grow problematic. Documented limitation.
- **Cross-machine portability**: state is local to the user account on each machine; sharing across machines is out of scope.

The `state/` directory is plugin-local user state (not shared, not Claude auto-memory) and uses markdown files for human inspectability. This is the *only* persistent state assemble-team writes outside the plan/recipe/report flow.

## Data flow

```
raw plan
  → enrich-plan        ── produces ──→  enriched plan markdown (frontmatter w/ ambiguity, classification, sources, forced_blocks_resolved)
  → verify-mapping     ── produces ──→  execution recipe markdown (frontmatter w/ recipe_id, mode, automation, guards_version, guards_hash)
  → execute-work       ── produces ──→  execution report markdown (frontmatter w/ recipe_id, result, pr_url, verdicts, notes)
```

All three handoffs are markdown with YAML frontmatter. The contracts below define tolerance and failure modes.

## Parsing & validation policy

LLM-produced markdown is forgiving in spirit but not always strict in form. Each sub-skill validates its input with the following layered policy:

### 1. Frontmatter extraction

- The first contiguous `---`...`---` block at the very top of the input is treated as YAML frontmatter. Leading blank lines and BOM are ignored.
- If the input is wrapped in a markdown code fence (```` ```markdown ... ``` ````), the outer fence is stripped before extraction. This is a common shape when users paste an LLM-formatted block.
- Trailing content after the closing `---` is treated as the body.
- Duplicate frontmatter blocks (two consecutive `---`...`---` segments at the very top, separated by zero or more blank lines and nothing else) → fatal: reject with "Multiple frontmatter blocks detected; remove the extra block." A `---` line that appears *inside* the body section (e.g., as a horizontal rule after non-trivial content) is not a frontmatter block and is ignored by this rule.
- Missing frontmatter → recoverable for `enrich-plan` (raw plans are expected to lack frontmatter); fatal for `verify-mapping` and `execute-work` (they expect upstream-produced markdown).

### 2. Required fields (fatal if missing)

| Sub-skill | Input required fields |
|---|---|
| `enrich-plan` | none — raw input accepted |
| `verify-mapping` | `forced_blocks_resolved: true`, non-empty `# Why` and `# Tasks` body sections |
| `execute-work` | `recipe_id`, `mode` (one of team/single/manual), `guards_version`, `guards_hash` |

Fatal validation failures produce an error message containing: (a) which field failed, (b) the canonical name and accepted aliases, (c) which sub-skill to run first to remediate. The raw input is **preserved** in the user-facing error so the user can edit and retry.

### 3. Alias normalization (recoverable — auto-normalized)

The parser accepts these aliases on input and normalizes to canonical form internally:

| Canonical | Accepted aliases |
|---|---|
| `pr_create` | `pr-create`, `prCreate` |
| `skip_qa` | `skip-qa`, `skipQa` |
| `allow_push` | `allow-push`, `allowPush` |
| `accept_edits` | `acceptEdits`, `accept-edits` |
| `read_only` | `read-only`, `readonly` |
| `mode: team` | `Team`, `TEAM` (case-insensitive) |

A normalization is **logged** to the consumer's nearest output `notes:` field, so the user can see what was rewritten:

- Normalization applied by `enrich-plan` (when accepting weird input) → emitted in the enriched plan's frontmatter `notes:` (a list field — see §"Sub-skill: `enrich-plan`" output schema for the field; added to the spec as part of this revision).
- Normalization applied by `verify-mapping` (when accepting aliases from the enriched plan or hand-authored input) → emitted in the execution recipe's frontmatter `notes:` list.
- Normalization applied by `execute-work` (when accepting aliases on the recipe) → emitted in the execution report's `notes:` list.

The `notes:` field on each artifact is a list of short strings (≤ 80 chars each); empty list permitted.

### 4. Cross-field invariants (fatal if violated)

| Invariant | Enforced by |
|---|---|
| `mode=team` requires at least one team card in the body | execute-work |
| `mode=single` requires exactly one card (single-agent spec) in the body | execute-work |
| `mode=manual` requires a non-empty manual prompt in the body | execute-work |
| `automation.pr_create=true` AND `mode=manual` → reject (manual cannot create PRs) | verify-mapping & execute-work |
| `automation.pr_create=true` AND `mode=single` AND `automation.skip_qa=false` → reject (no qa coverage) | verify-mapping & execute-work |
| `mode=team` AND `team_name` absent → fatal (every team run needs a stable team name) | verify-mapping & execute-work |
| `recipe_id` already present in `state/completed/` → emit `result: already_started`, no side effects | execute-work |
| `recipe_id` matches truth-table anomaly row (see Idempotency contract) | execute-work |
| `automation` object absent | treat as all-false defaults (`pr_create: false, allow_push: false, skip_qa: false`); no error |
| `automation` object present but contains an unknown nested key (e.g., `automation.merge: true`) | fatal: "Unknown automation key `merge`; supported: pr_create / allow_push / skip_qa" |
| `guards_hash` does not match the runtime canonical hash of the guards-version bundle | fatal: "Guard set drifted between verify-mapping and execute-work; re-run verify-mapping" |

### 4a. Guards hash specification

- The canonical guard set is the **full ordered list of guard names** (not bodies) in v0.1.0 `ROLES.md` § "Common guards", numbered 1–8 — *always all 8*, regardless of which mode the recipe is for.
- `guards_hash` is computed over this full bundle even when the executing mode injects only a subset (e.g., mode=single omits guard 8). The hash is a property of the *bundle the plugin ships*, not of the per-mode injection. The omission of guard 8 in mode=single is recorded in the execution report's `notes:` field, but does not change `guards_hash`.
- `guards_hash` = the first 12 hex chars of `sha256(<joined-names>)`, where `<joined-names>` is the guard names joined by `"\n"` after sorting alphabetically. Sort makes the hash insensitive to ordering changes; the canonical names are stable for v0.2.0.
- Example: for the 8 v0.1.0 guards (`Boolean defaults`, `Branching`, `CI / deploy files`, `Commit convention`, `No auto-push / no auto-merge`, `Package manager respect`, `Secrets`, `Teammate spawn guard`), the hash is the first 12 hex of sha256 over those 8 strings joined by `\n` after sorting.
- `guards_version` is `"1.0.0"` for the v0.1.0 guard bundle (unchanged from v0.1.0 in this migration).

### 5. Per-sub-skill input validation tables (standalone invocation defense)

Each sub-skill, when invoked standalone, must classify weird inputs and respond with the table below:

**`enrich-plan`** input cases:

| Shape | Action |
|---|---|
| File path that exists, has plan-shaped content | `Read` it; treat as inline plan body |
| File path that does not exist | Fatal: "File not found" |
| URL (https://github.com/.../issues/N or .../pull/N) | Fetch via `gh ... --repo <owner>/<repo>`; on failure fall back to grilling |
| URL non-GitHub | `WebFetch`; on failure fall back to grilling |
| Inline plan body < 40 chars or no nouns | Reject: "Plan too thin; expand or use PLAN_TEMPLATE.md" |
| Already-enriched markdown (has frontmatter with `forced_blocks_resolved`) | Accept; re-score ambiguity; skip steps already passed |

**`verify-mapping`** input cases:

| Shape | Action |
|---|---|
| Enriched plan (frontmatter w/ `forced_blocks_resolved: true`) | Accept, proceed |
| Raw plan (no frontmatter, prose body) | Reject: "Run /enrich-plan first" + offer to chain through orchestrator |
| Execution recipe (mode field present, looks like verify-mapping output) | Reject: "This is already a recipe; run /execute-work" + suggest |
| Frontmatter present but `forced_blocks_resolved: false` | Reject: "Plan has unresolved gaps; re-run /enrich-plan" |
| File path | `Read` it; classify by shape |

**`execute-work`** input cases:

| Shape | Action |
|---|---|
| Execution recipe (frontmatter w/ `recipe_id` + `mode`) | Accept, proceed |
| Enriched plan (no `mode`) | Reject: "Not a recipe; run /verify-mapping first" |
| Raw plan | Reject: "Not a recipe; run /enrich-plan then /verify-mapping" |
| Recipe with `mode=team` but zero team cards | Fatal: "Recipe inconsistent: mode=team requires at least one card" |
| Recipe with `mode=single` and 2+ cards | Fatal: "Recipe inconsistent: mode=single allows exactly one card" |
| Recipe with `recipe_id` already in `started/` not in `completed/` | Halt, ask resume/restart/abort (idempotency rule) |
| File path | `Read` it; classify by shape |

In every reject case, the message names (a) the detected shape, (b) the expected shape, (c) the remediation command to run.

## Slash commands

Each `commands/<name>.md` is a thin shim that invokes its corresponding skill via the Skill tool. The shim is necessary because Claude Code skill triggering relies on the skill's frontmatter description matching the user's intent — explicit slash commands give predictable, discoverable entry points.

### Command file format (resolved)

Each command file uses Claude Code's standard command frontmatter (`description`, `argument-hint`, `allowed-tools`) and the `$ARGUMENTS` placeholder for runtime input. Example for `commands/enrich-plan.md`:

```markdown
---
description: Enrich a plan with classification + ambiguity scoring + grilling (step 1 of assemble-team)
argument-hint: <plan body | path to plan file | GitHub issue/PR URL>
allowed-tools: Skill, Read, Glob, Grep, Bash, AskUserQuestion, WebFetch
---

Invoke the `enrich-plan` skill from the assemble-team plugin on the plan body, path, or URL provided in the arguments. Pass the user input through verbatim:

$ARGUMENTS
```

The full set of four commands, their `argument-hint`, and their `allowed-tools`:

| Command | `argument-hint` | `allowed-tools` |
|---|---|---|
| `/assemble-team` | `<plan body \| path to plan file \| GitHub issue/PR URL>` | `Skill, Read, Glob, Grep, Bash, AskUserQuestion, WebFetch, Agent, TeamCreate, TeamDelete, TaskList, SendMessage` |
| `/enrich-plan` | `<plan body \| path to plan file \| GitHub issue/PR URL>` | `Skill, Read, Glob, Grep, Bash, AskUserQuestion, WebFetch` |
| `/verify-mapping` | `<enriched plan markdown \| path to enriched plan file>` | `Skill, Read, Glob, Grep, AskUserQuestion, Bash` |
| `/execute-work` | `<execution recipe markdown \| path to recipe file>` | `Skill, Read, Glob, Grep, Bash, AskUserQuestion, Agent, TeamCreate, TeamDelete, TaskList, SendMessage` |

`TaskList` is included in `/assemble-team` and `/execute-work` because the resume semantics of the idempotency contract use `TaskList` to enumerate live team task status.

The shim body is intentionally minimal — the Skill tool loads the corresponding `skills/<name>/SKILL.md` and that file owns the workflow.

## Manifest changes

### `plugins/assemble-team/.claude-plugin/plugin.json`

```json
{
  "name": "assemble-team",
  "version": "0.2.0",
  "description": "5-layer harness (Enrich → Map → Execute) that converts a plan into a safely-spawned agent team, a single subagent, or a printed prompt. Three sub-skills callable independently or chained via /assemble-team. Universal — works with any monorepo.",
  "author": { "name": "skarl86", "url": "https://github.com/skarl86" },
  "license": "MIT",
  "homepage": "https://github.com/skarl86/claude-plugins/tree/main/plugins/assemble-team",
  "repository": "https://github.com/skarl86/claude-plugins"
}
```

### `.claude-plugin/marketplace.json` (monorepo root)

Add to `plugins` array:

```json
{
  "name": "assemble-team",
  "source": "./plugins/assemble-team",
  "description": "5-layer harness (Enrich → Map → Execute) that converts a plan into a safely-spawned agent team, a single subagent, or a printed prompt. Three sub-skills callable independently or chained via /assemble-team."
}
```

### `README.md` (monorepo root)

Add a `### [assemble-team](plugins/assemble-team)` section under `## Plugins`, matching the tone and length of the four existing plugin entries. Mention: 3 sub-skills + 4 slash commands + mode branching (team/single/manual).

## Error handling

### New (sub-skill chain)

| Situation | Handling |
|---|---|
| Sub-skill returns invalid output (frontmatter missing required fields) | Orchestrator reports to user, halts chain, preserves partial output |
| User aborts at any AskUserQuestion gate | Halt, preserve partial output, note resume point |
| `verify-mapping` receives non-enriched plan | Reject with "Run enrich-plan first" + offer to chain automatically through the orchestrator |
| `execute-work` receives invalid recipe | Reject with format error + missing-field list (preserves raw input in the error so the user can edit and retry) |
| `mode=single` selected but recipe has 2+ writer cards | Block silent pruning. Require explicit `AskUserQuestion` confirming (a) which cards are dropped, (b) coverage lost, (c) any required automation flag changes (e.g., `skip_qa`) |
| `mode=team` requested but `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS` unset or `teammateMode` ≠ tmux | verify-mapping recommends `single` if the mapping fits one writer; otherwise blocks with a remediation checklist (do **not** silently fall back to `manual` — that abandons execution) |
| `automation.pr_create=true` but `gh` CLI missing | Keep current mode (team/single), set `pr_create: false` in the recipe, note in `notes:` "pr_create disabled: gh CLI not found"; emit remediation hint ("install gh + auth, or accept no auto-PR for this run"). User confirms once before recipe emits |
| Recipe re-submitted with already-started `recipe_id` | Halt before any side effect; ask resume / restart / abort (see Idempotency contract). Default reply on user silence = abort |
| Frontmatter parse failure (malformed YAML) | Fatal with raw-input preserved; suggest "re-emit upstream sub-skill or hand-edit between the `---` markers" |
| Alias on input field (e.g., `pr-create` instead of `pr_create`) | Recoverable; auto-normalize to canonical; log in `notes:` |
| Frontmatter wrapped in markdown code fence | Recoverable; strip outer fence; log in `notes:` |

### Preserved from v0.1.0

| Situation | Handling |
|---|---|
| GitHub issue/PR fetch fails | Fall back to grilling (ask user for Goal / Scope / Tasks directly) |
| Forced-block (Goal missing, Tasks==0, destructive + Constraints missing) | Block, ask user, no skip allowed |
| L4 mapping not approved | No spawn, halt |
| Teammate stuck | Report to user, ask "extra instruction / replace / shutdown" |
| `automation.pr_create: true` blocked by qa ❌ | No PR, report verdicts, ask user to fix and rerun |
| Worktree conflict / overlap | Auto-isolate or ask user for serialization decision |

## Testing strategy

No automated test framework (Claude Code plugin convention). Manual checklist:

### A. Regression — mode=team path (highest priority)

1. Run the `PLAN_TEMPLATE.md` refund-dialog example through `/assemble-team`. Expected: same team mapping, same PR-flow, same final PR (or block) as v0.1.0.

### B. New paths

2. **mode=single happy path**: a single-file rename plan → user picks single → one Agent call → report formatted correctly.
3. **mode=single + `pr_create: true` + `skip_qa: true`** (or aliased `pr-create` / `skip-qa` in the plan body): PR created via gh; the PR body must begin (after any leading whitespace) with the exact three-line blockquote specified in §"Mode=single / PR-flow":
   ```
   > **assemble-team mode=single**: this PR was produced by a single subagent.
   > No reviewer verdict and no QA verdict were collected.
   > The human PR reviewer is the only review gate.
   ```
   Test asserts byte-for-byte match.
4. **mode=single + `pr_create: true` without `skip_qa: true`**: blocked with clear remediation message naming the missing flag.
5. **mode=manual**: any plan, user picks manual → prompts printed, no spawn, `result: printed`.
6. **Auto-fallback (gh missing, execution still possible)**: simulate `gh` missing + plan requests PR auto-creation (using canonical `automation.pr_create: true` or alias `automation: pr-create`) → verify-mapping keeps the mode it would otherwise pick (team or single), sets `pr_create: false` in the recipe, and adds a `notes:` entry "pr_create disabled: gh CLI not found." It does NOT recommend manual — manual is only recommended when execution itself is infeasible (e.g., `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS` unset and only multi-writer mapping fits).
   - 6a. **Capability missing for team but single is feasible**: simulate `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS` unset + plan maps to 1 writer card → verify-mapping recommends `single`, not manual.
   - 6b. **Capability missing for both team and single**: simulate the above + plan maps to 2+ writer cards → verify-mapping blocks with a remediation checklist; manual is offered only as an explicit user opt-in.

### C. Contract / standalone

7. `/enrich-plan "<raw plan>"` standalone → enriched plan returned, halts.
8. `/verify-mapping "<enriched plan>"` standalone → recipe returned, halts.
9. `/verify-mapping "<raw plan without frontmatter>"` → rejected with guidance.
10. `/execute-work "<recipe>"` standalone → executes correctly.

### D. Migration integrity

11. **Empty diff** of `ROLES.md`, `GRILL_PLAN.md`, `PLAN_TEMPLATE.md` against the v0.1.0 originals at `~/.claude/plugins/marketplaces/skarl86-claude-plugins/plugins/assemble-team/skills/assemble-team/`. Verbatim means verbatim — no exceptions, no frontmatter adjustments, no path rewrites in the body text. (Any path references inside those files use relative names like `ROLES.md` / `GRILL_PLAN.md` / `PLAN_TEMPLATE.md` which still resolve correctly within the new `skills/enrich-plan/` directory; `ROLES.md` references from new SKILL.md files use the new layout — i.e., the *callers* change, not the reference docs.)
12. All 8 common guards present in spawn prompts (mode=team). Verify with `guards_version` and `guards_hash` matching the canonical set.
13. All automation flags (pr-create / skip-qa / allow-push) work identically — both via v0.1.0's hyphenated form (alias-accepted) and the new canonical snake_case form.

### E. User-flow smoke

14. Happy path: well-formed plan → low ambiguity → team mode → success.
15. High ambiguity: grilling fires; user fills; chain proceeds.
16. User aborts at L4 verification gate → no spawn, partial output preserved.

### F. Parsing & validation defense

17. Invoke `/verify-mapping` with an **enriched plan** wrapped in a markdown code fence (```` ```markdown\n---\n...\n``` ````) → outer fence stripped, parsing proceeds, normalization "outer code fence stripped" appears in `notes:` of the recipe output.
18. Invoke `/execute-work` with a recipe that uses hyphenated nested aliases:
    ```yaml
    automation:
      pr-create: true
      skip-qa: true
    ```
    → both aliases normalized to `pr_create` / `skip_qa`; both normalizations logged in the report's `notes:`. (This shape IS valid YAML — only the keys are aliased.)
19. Invoke `/verify-mapping` with two consecutive `---`...`---` blocks at top → fatal "Multiple frontmatter blocks", raw preserved.
20. Invoke `/execute-work` with valid frontmatter but body missing `# Team cards` while `mode=team` → fatal "Recipe inconsistent: mode=team requires at least one card".
21. Invoke `/verify-mapping` with a payload that is **already an execution recipe** (frontmatter has `mode:` and `recipe_id:`) → reject with "This input is already a recipe; run /execute-work instead." This is distinct from test 17 — fence-stripping precedes the shape check, but a payload identified as a recipe (regardless of fencing) is rejected by verify-mapping.
22. **Completed-recipe re-invocation**: invoke `/execute-work` with the same `recipe_id` twice (second call after first completes successfully) → second returns `result: already_started` without side effects, references the original `completed/<recipe_id>.md` summary.
23. **Incomplete-recipe resume**: simulate `execute-work` mid-run failure (kill the session before completion), then re-invoke with the same `recipe_id` → `started/<recipe_id>.md` exists, `completed/<recipe_id>.md` and `aborted/<recipe_id>.md` do not → prompts user to resume / restart / abort.
24. **User-aborted recipe**: simulate user picking abort in test 23 → sentinel moves from `started/` to `aborted/` → re-invoking `/execute-work` with the same `recipe_id` is blocked with "This recipe_id was aborted previously and cannot be reused. Re-run /verify-mapping to obtain a new recipe_id."
25. **Restart after abort uses a NEW recipe_id**: confirm test 24's behavior is correct — there is no "restart-after-abort" transition; the user must run verify-mapping again to obtain a fresh recipe_id. This is documented behavior, not a bug.
26. **Atomic claim race**: spawn two `/execute-work` sessions on the same recipe_id within ~1 second of each other → only one wins the `set -C` claim and creates a flat-file `started/<recipe_id>.md`; the other observes the flat file already present and follows the resume/restart/abort prompt. Assert that `started/<recipe_id>.md` is a regular file (not a directory).
27. **Sentinel anomaly**: manually copy `started/<id>.md` to also exist in `completed/<id>.md` (simulating a single-location invariant violation), then `/execute-work` with that recipe_id → fatal "Anomaly: multiple state sentinels for recipe_id; inspect ~/.claude/plugins/assemble-team/state/".
27a. **Partial-write recovery**: manually create a 0-byte `started/<id>.md` (simulates a process killed between claim and body write) and a stale `.${id}.md.<pid>.tmp` containing partial body. Invoke `/execute-work` with the same `recipe_id` → lookup sees flat `started/<id>.md` (size 0 acceptable), ignores the stale `.tmp` (lookup uses exact-path test on `<id>.md`), and the user is prompted resume / restart / abort as for any incomplete run. The stale temp remains on disk (inert) until manual cleanup; this is documented behavior, not a bug.
27b. **No orphan-directory residue (code-review invariant check)**: this is a static-analysis assertion, not a runtime test. Verified by grepping the execute-work skill's SKILL.md and any associated scripts for `mkdir.*lock`, `.lock/`, or directory-mode sentinel creation patterns. The check passes if and only if zero such patterns are found. (Runtime tests pass vacuously regardless — only code review catches a regression here.)
28. Invoke `/execute-work` with a recipe containing an unknown automation key (`automation.merge: true`) → fatal "Unknown automation key `merge`; supported: pr_create / allow_push / skip_qa".
29. Invoke `/execute-work` with a recipe whose `guards_hash` differs from the runtime canonical hash → fatal "Guard set drifted between verify-mapping and execute-work; re-run verify-mapping".

## Migration order (implementation hint)

This is not the implementation plan (see writing-plans skill), but the suggested file-creation order for reviewer mental model:

1. `plugin.json` (v0.2.0)
2. `skills/enrich-plan/SKILL.md` (L1+L2+L3 consolidated)
3. `skills/enrich-plan/GRILL_PLAN.md` (verbatim move from v0.1.0)
4. `skills/enrich-plan/PLAN_TEMPLATE.md` (verbatim move from v0.1.0)
5. `skills/verify-mapping/SKILL.md` (L4 + mode selection NEW; includes the deterministic precedence table)
6. `skills/verify-mapping/ROLES.md` (verbatim move from v0.1.0)
7. `skills/execute-work/SKILL.md` (L5 split into team/single/manual NEW; includes the idempotency check using `~/.claude/plugins/assemble-team/state/`)
8. `skills/assemble-team/SKILL.md` (orchestrator NEW, thin — validates handoffs, owns halt/resume)
9. `commands/assemble-team.md` (slash shim with canonical frontmatter)
10. `commands/enrich-plan.md` (slash shim)
11. `commands/verify-mapping.md` (slash shim)
12. `commands/execute-work.md` (slash shim)
13. `README.md` at plugin root (updated)
14. `marketplace.json` entry at monorepo root
15. `README.md` at monorepo root (new plugin section)

The `state/` directory at `~/.claude/plugins/assemble-team/state/{started,completed,aborted}/` is created lazily by `execute-work` on first run — no need to seed it during install. (See §"Idempotency contract" for the full lifecycle of each subdirectory.)

## Deferred to a future version

The brainstorming session explicitly considered and deferred:

- **Form A — Team Preset**: rejected as wrong abstraction (team composition is plan-dependent; presets lock users into specific shapes).
- **Form B — Project Override (`.claude/assemble-team.local.md`)**: custom roles, guard overrides, default constraints. Deferred for unclear value vs. implementation cost at this time; may revisit when concrete pain points emerge.
- **Form C — Auto-Memory / pattern detection**: deferred for high cost and risk of false recall.
- **Form D — User Config**: deferred along with B/C.
- **Hooks** (SessionStart, PreToolUse, etc.): deferred; not required for structural split.
- **Agents** (Agent file definitions): deferred; mode=single uses the runtime Agent tool, not packaged definitions.

These remain candidates for v0.3.0+ but are intentionally out of scope here to keep this work focused on the migration + structural split + mode branching.
