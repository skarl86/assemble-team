# assemble-team v0.2.0 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Migrate the `assemble-team` plugin from its external source into this monorepo and split its single skill into three independently callable sub-skills (`enrich-plan` / `verify-mapping` / `execute-work`) plus a thin orchestrator (`assemble-team`), with new mode branching (team / single / manual) and idempotency state tracking.

**Architecture:** A v0.1.0 → v0.2.0 monorepo migration with structural decomposition. The 5-layer harness (L1 Entry → L2 Routing → L3 Enrichment → L4 Verification → L5 Handoff) is preserved as the underlying logic but redistributed: L1+L2+L3 → `enrich-plan`, L4 + new mode selection → `verify-mapping`, L5 with team/single/manual branches → `execute-work`. The orchestrator chains them. Reference docs (ROLES.md / GRILL_PLAN.md / PLAN_TEMPLATE.md) move verbatim. Slash command shims provide entry points. State directory (`~/.claude/plugins/assemble-team/state/`) provides idempotency.

**Tech Stack:** Claude Code plugin spec (skills/commands/manifest), Markdown + YAML frontmatter, bash (atomic claim mechanism uses `set -C` noclobber).

**Source of truth:** Design spec at `docs/superpowers/specs/2026-05-19-assemble-team-v2-design.md` (785 lines, 6-round Codex-reviewed, implementation-ready).

**v0.1.0 source location** (read-only reference for verbatim moves and L1/L2/L3/L4/L5 content):
- `~/.claude/plugins/marketplaces/skarl86-claude-plugins/plugins/assemble-team/`

---

## Phase 0 — Worktree & directory scaffolding

### Task 0: Create plugin directory layout

**Files:**
- Create: `plugins/assemble-team/.claude-plugin/` (directory)
- Create: `plugins/assemble-team/commands/` (directory)
- Create: `plugins/assemble-team/skills/assemble-team/` (directory)
- Create: `plugins/assemble-team/skills/enrich-plan/` (directory)
- Create: `plugins/assemble-team/skills/verify-mapping/` (directory)
- Create: `plugins/assemble-team/skills/execute-work/` (directory)

- [ ] **Step 1: Create the plugin's directory tree**

Run from the worktree root (`/home/namgee/Development/private/claude-plugins/.claude/worktrees/assemble-team-v2-design/`):

```bash
mkdir -p plugins/assemble-team/.claude-plugin
mkdir -p plugins/assemble-team/commands
mkdir -p plugins/assemble-team/skills/assemble-team
mkdir -p plugins/assemble-team/skills/enrich-plan
mkdir -p plugins/assemble-team/skills/verify-mapping
mkdir -p plugins/assemble-team/skills/execute-work
```

- [ ] **Step 2: Verify the layout matches the spec**

Run:

```bash
find plugins/assemble-team -type d | sort
```

Expected output (six directories):

```
plugins/assemble-team
plugins/assemble-team/.claude-plugin
plugins/assemble-team/commands
plugins/assemble-team/skills
plugins/assemble-team/skills/assemble-team
plugins/assemble-team/skills/enrich-plan
plugins/assemble-team/skills/execute-work
plugins/assemble-team/skills/verify-mapping
```

- [ ] **Step 3: Commit the empty scaffold**

`git` does not track empty directories; this commit will land once subsequent tasks populate files. Skip the commit here — the directory layout is recorded by the first file commit in later tasks.

---

## Phase 1 — Plugin manifest

### Task 1: Write `plugins/assemble-team/.claude-plugin/plugin.json` (v0.2.0)

**Files:**
- Create: `plugins/assemble-team/.claude-plugin/plugin.json`

- [ ] **Step 1: Write the manifest with v0.2.0 metadata**

Create `plugins/assemble-team/.claude-plugin/plugin.json` with this exact content:

```json
{
  "name": "assemble-team",
  "version": "0.2.0",
  "description": "5-layer harness (Enrich → Map → Execute) that converts a plan into a safely-spawned agent team, a single subagent, or a printed prompt. Three sub-skills callable independently or chained via /assemble-team. Universal — works with any monorepo.",
  "author": {
    "name": "skarl86",
    "url": "https://github.com/skarl86"
  },
  "license": "MIT",
  "homepage": "https://github.com/skarl86/claude-plugins/tree/main/plugins/assemble-team",
  "repository": "https://github.com/skarl86/claude-plugins"
}
```

- [ ] **Step 2: Validate the JSON parses**

Run:

```bash
python3 -m json.tool plugins/assemble-team/.claude-plugin/plugin.json > /dev/null && echo OK
```

Expected output: `OK`. (If `python3` is unavailable, substitute `jq . plugins/assemble-team/.claude-plugin/plugin.json > /dev/null && echo OK`.)

- [ ] **Step 3: Commit**

```bash
git add plugins/assemble-team/.claude-plugin/plugin.json
git commit -m "feat(assemble-team): add v0.2.0 plugin manifest"
```

---

## Phase 2 — Verbatim migration of reference docs

This phase moves three reference markdown files from v0.1.0 without modification. The design spec requires an empty diff against the original.

### Task 2: Verbatim copy of `GRILL_PLAN.md` into `enrich-plan/`

**Files:**
- Create: `plugins/assemble-team/skills/enrich-plan/GRILL_PLAN.md`
- Source: `~/.claude/plugins/marketplaces/skarl86-claude-plugins/plugins/assemble-team/skills/assemble-team/GRILL_PLAN.md`

- [ ] **Step 1: Copy the file**

```bash
cp ~/.claude/plugins/marketplaces/skarl86-claude-plugins/plugins/assemble-team/skills/assemble-team/GRILL_PLAN.md \
   plugins/assemble-team/skills/enrich-plan/GRILL_PLAN.md
```

- [ ] **Step 2: Verify byte-identical**

```bash
diff -q ~/.claude/plugins/marketplaces/skarl86-claude-plugins/plugins/assemble-team/skills/assemble-team/GRILL_PLAN.md \
        plugins/assemble-team/skills/enrich-plan/GRILL_PLAN.md && echo "identical"
```

Expected: `identical` (no diff output). If anything else, abort and investigate.

- [ ] **Step 3: Commit**

```bash
git add plugins/assemble-team/skills/enrich-plan/GRILL_PLAN.md
git commit -m "feat(assemble-team): verbatim move GRILL_PLAN.md into enrich-plan sub-skill"
```

### Task 3: Verbatim copy of `PLAN_TEMPLATE.md` into `enrich-plan/`

**Files:**
- Create: `plugins/assemble-team/skills/enrich-plan/PLAN_TEMPLATE.md`
- Source: `~/.claude/plugins/marketplaces/skarl86-claude-plugins/plugins/assemble-team/skills/assemble-team/PLAN_TEMPLATE.md`

- [ ] **Step 1: Copy the file**

```bash
cp ~/.claude/plugins/marketplaces/skarl86-claude-plugins/plugins/assemble-team/skills/assemble-team/PLAN_TEMPLATE.md \
   plugins/assemble-team/skills/enrich-plan/PLAN_TEMPLATE.md
```

- [ ] **Step 2: Verify byte-identical**

```bash
diff -q ~/.claude/plugins/marketplaces/skarl86-claude-plugins/plugins/assemble-team/skills/assemble-team/PLAN_TEMPLATE.md \
        plugins/assemble-team/skills/enrich-plan/PLAN_TEMPLATE.md && echo "identical"
```

Expected: `identical`.

- [ ] **Step 3: Commit**

```bash
git add plugins/assemble-team/skills/enrich-plan/PLAN_TEMPLATE.md
git commit -m "feat(assemble-team): verbatim move PLAN_TEMPLATE.md into enrich-plan sub-skill"
```

### Task 4: Verbatim copy of `ROLES.md` into `verify-mapping/`

**Files:**
- Create: `plugins/assemble-team/skills/verify-mapping/ROLES.md`
- Source: `~/.claude/plugins/marketplaces/skarl86-claude-plugins/plugins/assemble-team/skills/assemble-team/ROLES.md`

- [ ] **Step 1: Copy the file**

```bash
cp ~/.claude/plugins/marketplaces/skarl86-claude-plugins/plugins/assemble-team/skills/assemble-team/ROLES.md \
   plugins/assemble-team/skills/verify-mapping/ROLES.md
```

- [ ] **Step 2: Verify byte-identical**

```bash
diff -q ~/.claude/plugins/marketplaces/skarl86-claude-plugins/plugins/assemble-team/skills/assemble-team/ROLES.md \
        plugins/assemble-team/skills/verify-mapping/ROLES.md && echo "identical"
```

Expected: `identical`.

- [ ] **Step 3: Commit**

```bash
git add plugins/assemble-team/skills/verify-mapping/ROLES.md
git commit -m "feat(assemble-team): verbatim move ROLES.md into verify-mapping sub-skill"
```

---

## Phase 3 — Sub-skill SKILL.md files (new content)

Four new `SKILL.md` files are created. Each is the main entry the Skill tool loads for that sub-skill. They draw content from both v0.1.0 (preserved behaviors) and the new design spec (mode branching, idempotency, parsing policy, frontmatter contracts).

### Task 5: Write `skills/enrich-plan/SKILL.md`

**Files:**
- Create: `plugins/assemble-team/skills/enrich-plan/SKILL.md`
- Reference: spec §"Sub-skill: `enrich-plan`" (lines ~97–158), §"Parsing & validation policy" (lines ~450–540)
- Reference: v0.1.0 SKILL.md § L1 Entry, § L2 Routing, § L3 Enrichment (lines 37–131)

- [ ] **Step 1: Write the file**

Create `plugins/assemble-team/skills/enrich-plan/SKILL.md` with this content:

````markdown
---
name: enrich-plan
description: |
  Step 1 of assemble-team. Accept any plan-shaped input (file path, GitHub
  issue/PR URL, or inline markdown), classify it by intent × complexity,
  fill its gaps via ambiguity scoring + grilling, and emit an enriched plan
  with source-tagged fields and a YAML frontmatter contract that downstream
  skills can validate.

  Trigger when the user says:
  - "/enrich-plan ..." (slash command)
  - "enrich this plan" / "이 plan 점수 매겨" / "fill in the gaps in this plan"
  - The orchestrator (assemble-team skill) invokes this as the first step

  Do NOT use this skill to spawn a team or call TeamCreate — that is
  execute-work's job. This skill only produces an enriched plan markdown.
---

# enrich-plan

Step 1 of the assemble-team v0.2.0 chain. Covers L1 Entry, L2 Routing, and L3 Enrichment of the original 5-layer harness.

## What this skill does

Given a raw plan (path / URL / inline markdown), produce an **enriched plan markdown** with:

- A YAML frontmatter block carrying ambiguity score, intent × complexity classification, per-field source tags, and a `forced_blocks_resolved: true|false` flag.
- A body with sections: `# Why`, `# Scope`, `# Tasks`, `# Constraints`, `# Definition of Done`, `# Out of scope`, `# Risks` — each line tagged with its source (`[from-user]` / `[from-code: path:line]` / `[guess]`).

The downstream skill `verify-mapping` reads this output and derives an execution recipe.

## Output contract (what verify-mapping expects)

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

## L1 · Input loading

Accept one of the following forms. Classify on input shape and dispatch:

| Shape | Action |
|---|---|
| File path that exists, has plan-shaped content | `Read` it; treat as inline plan body |
| File path that does not exist | Fatal: "File not found: `<path>`" |
| URL `https://github.com/<owner>/<repo>/issues/<num>` or `.../pull/<num>` | Parse the URL to extract owner/repo/num explicitly. Fetch via `gh issue view <num> --repo <owner>/<repo> --json title,body,comments` (or `gh pr view ...`). Expand into the plan body under a `## (original URL)` header. Do NOT assume the current working directory's repo. |
| URL non-GitHub | `WebFetch`; on failure fall back to grilling |
| Inline plan body | Use directly |
| Inline plan body < 40 chars or no nouns | Reject: "Plan too thin; expand or use PLAN_TEMPLATE.md" |
| Already-enriched markdown (frontmatter has `forced_blocks_resolved`) | Accept; re-score ambiguity; skip steps already passed |

On GitHub fetch failure (network error, auth missing, etc.), fall back to grilling: ask the user for Goal / Scope / Tasks directly.

If both prose and URLs are present in the body, treat the prose as authoritative and URLs as references the lead may fetch on demand.

### Pre-gate (block invalid invocations)

- No plan + no slash arg + no pasted body → respond with `PLAN_TEMPLATE.md` excerpt and ask the user to provide a plan.
- Plan body shorter than ~40 characters or no nouns → treat as invalid; ask user to expand.

## L2 · Classification & tool selection

Read `verify-mapping/ROLES.md` (sibling sub-skill's reference file) for the canonical role catalog and intent signal table. (`enrich-plan` does not derive team cards; it only classifies the plan.)

### Axis 1 — Intent

Pick one of {Trivial, Refactor, Build, Mid-sized, Collaborative, Architecture, Research} using the keyword signals listed in `verify-mapping/ROLES.md` § "Intent classification signals."

When multiple intents could apply, pick the one that best characterizes the work *as a whole*. When none match, ask the user one targeted question: "Which best fits — ⟨top-2 intents⟩?"

### Axis 2 — Complexity

- **Trivial** — single file, < 10 LoC, no dependency impact
- **Simple** — 1-2 files, scoped, < 30 min equivalent
- **Complex** — 3+ files OR architectural impact

### Tool-selection consequences for L3

- Trivial → grill only forced-block items (Goal / Tasks / destructive-without-Constraints).
- Simple → grill only failing dimensions.
- Complex → full ambiguity walkthrough.
- Build / Architecture → run `Glob` / `Read` / `Grep` to map existing patterns and cite paths in the enriched plan's `# Scope` and `# Risks` sections.
- Research → local read-only scans with distinct exit criteria per probe; include findings in the ambiguity summary.

**Spawn discipline (core safety guarantee).** This skill MUST NOT call `TeamCreate`, `Agent`, or spawn any teammate. All exploration is local, read-only, and performed by this skill using `Glob` / `Read` / `Grep`.

## L3 · Enrichment

Follow the procedure in `GRILL_PLAN.md` (sibling reference file). Summary:

1. Score ambiguity across 5 dimensions (Goal / Scope / Tasks / Constraints / DoD), each 0.0–1.0.
2. `Clarity = (Goal + Scope + Tasks + Constraints + DoD) / 5`; `Ambiguity = 1 - Clarity`.
3. If `Ambiguity ≤ 0.30` and no forced-block trigger, skip grilling; proceed to output.
4. Otherwise enter grilling: one question at a time, recommendation provided, dependency order Why → Scope → Tasks → Constraints → DoD → Out-of-scope → Risks.
5. Prefer codebase exploration (`Glob` / `Read` / `Grep`) over asking the user when the answer is discoverable.
6. Attach a source tag (`[from-user]` / `[from-code: path:line]` / `[guess]`) to every filled field.

### Forced-block heuristics (binary — block even with passing score)

- `Goal` missing
- `Tasks` count == 0
- `Constraints` missing AND plan contains destructive signals (migration / drop / force-push)

When any of the above fire, set `forced_blocks_resolved: false` in the output frontmatter and refuse to emit — re-enter grilling until resolved.

## Normalization log (notes field)

When input contains user-facing aliases that get normalized to canonical form (e.g., `automation: pr-create` → `automation.pr_create: true`), record each normalization as a `<=80-char` string in the `notes:` list of the output frontmatter. Empty list permitted.

## Output validation before emitting

Before returning to the caller (the orchestrator or standalone user), check:

- `forced_blocks_resolved: true` (otherwise re-enter grilling)
- `guesses_to_confirm` list is empty OR contains only fields the user explicitly accepted as guesses
- All 7 body sections (Why / Scope / Tasks / Constraints / DoD / Out of scope / Risks) are non-empty

If any check fails, do not emit a malformed enriched plan — surface the failure to the caller.

## When NOT to use

- The plan is already enriched (use `verify-mapping` next).
- The user wants to spawn a team directly without classification (use `execute-work` if they have a recipe).
- Single-task work that needs no classification (use a normal subagent).
````

- [ ] **Step 2: Verify the file has valid frontmatter**

```bash
head -25 plugins/assemble-team/skills/enrich-plan/SKILL.md
```

Expected: starts with `---`, contains `name: enrich-plan` and `description: |`, ends frontmatter with another `---`.

- [ ] **Step 3: Commit**

```bash
git add plugins/assemble-team/skills/enrich-plan/SKILL.md
git commit -m "feat(assemble-team): add enrich-plan sub-skill SKILL.md (L1+L2+L3)"
```

### Task 6: Write `skills/verify-mapping/SKILL.md`

**Files:**
- Create: `plugins/assemble-team/skills/verify-mapping/SKILL.md`
- Reference: spec §"Sub-skill: `verify-mapping`" (lines ~160–225), §"Mode selection — deterministic precedence" (lines ~218–245), §"`mode=single` scope-reduction flow" (lines ~247–260)
- Reference: v0.1.0 SKILL.md § L4 Verification (lines 133–160)

- [ ] **Step 1: Write the file**

Create `plugins/assemble-team/skills/verify-mapping/SKILL.md` with this content:

````markdown
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
````

- [ ] **Step 2: Verify the file has valid frontmatter**

```bash
head -25 plugins/assemble-team/skills/verify-mapping/SKILL.md
```

Expected: starts with `---`, contains `name: verify-mapping`.

- [ ] **Step 3: Commit**

```bash
git add plugins/assemble-team/skills/verify-mapping/SKILL.md
git commit -m "feat(assemble-team): add verify-mapping sub-skill SKILL.md (L4 + mode selection)"
```

### Task 7: Write `skills/execute-work/SKILL.md`

**Files:**
- Create: `plugins/assemble-team/skills/execute-work/SKILL.md`
- Reference: spec §"Sub-skill: `execute-work`" (lines ~226–320), §"Idempotency contract" (lines ~340–460)
- Reference: v0.1.0 SKILL.md § L5 Handoff (lines 162–229)

- [ ] **Step 1: Write the file**

Create `plugins/assemble-team/skills/execute-work/SKILL.md` with this content:

````markdown
---
name: execute-work
description: |
  Step 3 of assemble-team. Accept an execution recipe (from verify-mapping
  or hand-authored), check the idempotency state for the recipe_id, and
  execute per the recipe's mode field:
    team   → TeamCreate + monitoring + optional PR-flow
    single → Agent tool (subagent_type=general-purpose), one subagent
    manual → print spawn prompt(s); do not execute
  Emit an execution report with the result and any PR URL.

  Trigger when the user says:
  - "/execute-work ..." (slash command)
  - "run this recipe" / "이 recipe로 실행해"
  - The orchestrator invokes this as step 3 after verify-mapping

  Idempotency: re-running with the same recipe_id detects prior runs via
  state files at ~/.claude/plugins/assemble-team/state/ and returns
  already_started with no side effects when the recipe was completed.
---

# execute-work

Step 3 of the assemble-team v0.2.0 chain. Covers L5 Handoff, branched by mode.

## What this skill does

Given an execution recipe, execute the work per its `mode:` field. Three modes are supported: `team`, `single`, `manual`. Idempotency is enforced via a state directory; re-running with the same `recipe_id` after completion is a safe no-op.

## Input contract & validation

**Input**: execution recipe markdown (output of `verify-mapping` or hand-authored matching the format).

Pre-condition checks (fatal if any fails):

| Check | Failure response |
|---|---|
| `recipe_id` present in frontmatter | Reject: "Not a recipe. Run `/verify-mapping` first." |
| `mode` present and one of {team, single, manual} | Reject with the offending value listed |
| `guards_version` and `guards_hash` present | Reject: "Recipe missing guards metadata; re-emit from `/verify-mapping`." |
| `mode=team` AND `team_name` present | Reject fatally if missing |
| `mode=team` AND body has at least one team card | Reject: "Recipe inconsistent: mode=team requires at least one card" |
| `mode=single` AND body has exactly one single-agent spec | Reject if 0 or 2+ |
| `mode=manual` AND body has non-empty manual prompt | Reject if missing |
| `automation.pr_create=true` AND `mode=manual` | Reject: manual cannot create PRs |
| `automation.pr_create=true` AND `mode=single` AND `automation.skip_qa=false` | Reject: no qa coverage |
| `guards_hash` matches runtime canonical hash | Fatal: "Guard set drifted between verify-mapping and execute-work; re-run verify-mapping" |
| `automation` has unknown nested key | Fatal: "Unknown automation key `<key>`; supported: pr_create / allow_push / skip_qa" |
| `automation` object absent | Treat as all-false defaults; do not error |

Aliases (`pr-create`, `skip-qa`, `allow-push`, etc.) are accepted on input and normalized to canonical names. Log each normalization in the output report's `notes:` list.

## Output contract

```markdown
---
recipe_id: <stable id copied from the execution recipe>
mode: team | single | manual
team_name: <same value as the recipe's team_name>      # mode=team only; null otherwise
result: completed | failed | aborted | printed | already_started
pr_url: <https URL>                                    # when PR created
duration_seconds: <int>
verdicts:                                              # mode=team only; null for single/manual
  qa: all-pass | partial-pass | failed | null
  reviewer: { errors: N, warnings: N, ok: N } | null
notes: []                                              # <=80-char strings
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

## Idempotency contract

State directory: `~/.claude/plugins/assemble-team/state/{started,completed,aborted}/<recipe_id>.md`

### Single-location invariant

At most one of `{started/, completed/, aborted/}` contains `<recipe_id>.md` at any moment. All transitions move the sentinel atomically; no transition creates or leaves a second copy.

### Pre-execution lookup

Before any side effect (TeamCreate / Agent / gh push / writing the started/ entry), test each of the three locations for the exact path `<dir>/<recipe_id>.md` (no glob; dot-prefixed temp files are naturally ignored):

```bash
STATE="$HOME/.claude/plugins/assemble-team/state"
mkdir -p "$STATE/started" "$STATE/completed" "$STATE/aborted"

started=0; completed=0; aborted=0
[ -f "$STATE/started/${recipe_id}.md" ] && started=1
[ -f "$STATE/completed/${recipe_id}.md" ] && completed=1
[ -f "$STATE/aborted/${recipe_id}.md" ] && aborted=1
sum=$((started + completed + aborted))
```

Then:

| State | Action |
|---|---|
| `sum == 0` | Fresh run. Perform atomic claim (below). |
| `started == 1, others == 0` | Incomplete run. Ask user resume / restart / abort. On 60s silence default abort. |
| `completed == 1, others == 0` | Previously completed. Emit `result: already_started` with no side effects. |
| `aborted == 1, others == 0` | Recipe_id retired. Block with: "This recipe_id was aborted previously and cannot be reused. Re-run /verify-mapping to obtain a new recipe_id." |
| `sum >= 2` | Anomaly. Fatal: "Anomaly: multiple state sentinels for recipe_id; inspect $STATE/". |

### Atomic claim mechanism (noclobber + temp-rename)

`set -C` (noclobber) is POSIX (POSIX.1-2017 §2.9.2). Per-process temp filename prevents shared-name races. Concrete sequence:

```bash
SENTINEL="$STATE/started/${recipe_id}.md"
# Per-process unique temp — $$ is the shell PID; concurrent claimants for the
# same recipe_id never share a temp filename.
TMP="$STATE/started/.${recipe_id}.md.$$.tmp"

# Step 1: atomic claim. Subshell scopes noclobber — do NOT flatten.
if ! ( set -C; : > "$SENTINEL" ) 2>/dev/null; then
  # File already exists — claim failed; follow lookup-present path.
  exit_with_lookup_present
fi

# Step 2: write body via temp-then-rename for partial-write safety.
printf '%s\n' "$body" > "$TMP"
mv "$TMP" "$SENTINEL"
```

Sentinel body fields (mode=team example):

```markdown
recipe_id: rcp-20260519-143012-a1b2c3d4
mode: team
team_name: team-refund-dialog-20260519-1430
started_at: 2026-05-19T14:30:12Z
plan_title: refund-dialog copy refresh
owned_scopes:
  - apps/web/src/components/RefundDialog.tsx
```

### Transitions

- **Successful completion**: `mv started/<id>.md completed/<id>.md`.
- **Terminal failure** (not user-aborted): `mv started/<id>.md completed/<id>.md` (body records the failure).
- **User abort**: `mv started/<id>.md aborted/<id>.md`. Recipe_id is retired; no restart from this id.
- **User restart**: `rm started/<id>.md`, then fresh atomic claim, then proceed.

### Resume semantics (mode=team only)

When user picks resume:
- Read `team_name` from `started/<recipe_id>.md`.
- Use the `TaskList` tool to enumerate live team task status.
- Skip any teammate whose task has reached its DoD (commit + report present).
- Re-spawn or extend any teammate not yet idle.

For mode=single and mode=manual, resume is unavailable. Answer: "single/manual mode does not support resume; choose restart or abort."

### Cleanup

Not auto-cleaned in v0.2.0. Stale temp files (`.${recipe_id}.md.<pid>.tmp`) from killed processes are inert (the lookup uses exact-path tests). Manual cleanup is `rm "$STATE"/{started,completed,aborted}/.*.tmp` and is safe at any time.

## Mode=team execution

Preserves v0.1.0 behavior. Use `TeamCreate` with `team_name` from the recipe. Each teammate spawn prompt receives, in order:

1. The teammate's role body — inline-injected from `verify-mapping/ROLES.md` (do NOT reference `subagent_type` for external subagents that restrict tools like `SendMessage`).
2. The 8 common domain guards from `ROLES.md` (referenced by `guards_version` / `guards_hash`).
3. The plan body (shared context).
4. The teammate's own task paragraph (from the recipe card).
5. Owned files / glob list.
6. Worktree path + branch (only when isolation is on).
7. Collaboration rules (other teammate names + when to `SendMessage` to whom).
8. Definition of Done (task-specific + common: Conventional Commits commit + one-paragraph report to lead via `SendMessage`).

### Worktree isolation

Default OFF. Switch ON when:
- The plan explicitly requests isolation.
- Two or more teammates would edit overlapping directories.
- The plan contains destructive operations.

When ON, `git worktree add` and pass the path + branch into the spawn prompt. On completion, do NOT auto-merge.

### Monitoring + automation gate

- Idle teammates notify the lead automatically.
- Stuck teammates: report to the user and ask "extra instruction / replace / shutdown".
- All-idle: produce a one-paragraph synthesis for the user.

### PR-flow (when `automation.pr_create: true`)

0. Confirm the flag is set explicitly by the user (present in original plan as `[from-user]`, or via explicit grilling yes). Lead-inferred natural-language signals ("create a PR for me") do NOT count.
1. qa verdict must be all green. Otherwise block and report.
   Exception: `automation.skip_qa: true` (with user-confirmed L3 yes) bypasses the qa gate. Record in `notes:`.
2. reviewer verdict — block on any `❌`; force a reviewer-summary banner at the top of the PR body if 1–2 `⚠️`; block on 3+ `⚠️`.
3. Discover default branch dynamically:

   ```bash
   DEFAULT_BRANCH=$(gh repo view --json defaultBranchRef --jq .defaultBranchRef.name)
   # Fallback:
   DEFAULT_BRANCH=$(git symbolic-ref refs/remotes/origin/HEAD | sed 's@^refs/remotes/origin/@@')
   ```

   3a. **Push reconciliation**: with `automation.allow_push: true` (or explicit user yes during L3), run:

   ```bash
   git push -u origin "$(git rev-parse --abbrev-ref HEAD)"
   ```

   Otherwise stop and ask once: "Branch `<name>` has not been pushed. Push and create the draft PR? (yes / no)" — proceed only on explicit yes.

   3b. **Non-interactive PR create**:

   ```bash
   PR_TITLE="<plan's Why first line, ≤70 chars>"
   PR_BODY_FILE=$(mktemp)
   # Synthesize body: plan Why + per-teammate report + qa verdict + reviewer banner
   printf '%s\n' "$pr_body" > "$PR_BODY_FILE"
   gh pr create \
     --draft \
     --base "$DEFAULT_BRANCH" \
     --head "$(git rev-parse --abbrev-ref HEAD)" \
     --title "$PR_TITLE" \
     --body-file "$PR_BODY_FILE"
   ```

   Both `--title` and `--body-file` MUST be present so `gh` does not drop into an interactive editor.

4. Never run `gh pr merge` automatically — always a human action.

### Cleanup

When user says "cleanup" / "팀 정리", call `TeamDelete`. Live teammates must be shut down before deletion.

## Mode=single execution

Exactly one code-modifying agent. No qa/reviewer/researcher.

Calls the runtime `Agent` tool with `subagent_type: "general-purpose"`. The ROLES.md spawn guard #8 (forbidding restrictive subagent_type for roles needing SendMessage) **does not apply** because single mode is non-collaborative by construction. Log this in `notes:`.

### Agent invocation prompt

Constructed inline (not by referring to an external agent file). Contains, in order:

1. **Single-mode preamble** (verbatim — this exact text):

   ```
   You are running in mode=single. There is no team, no other teammates, no SendMessage, no TaskUpdate signaling, and no inter-agent coordination. Any references to "the lead", "teammates", "SendMessage", "TaskUpdate", or "collaboration" inside the role body below DO NOT apply to you. Where the role body asks you to "report to the lead via SendMessage", interpret this as: include the same content as your final reply text.
   ```

2. The role body inline-injected from `verify-mapping/ROLES.md` (verbatim, no edits).
3. Common guards 1–7 (omit guard 8 — log "guard 8 omitted: single mode is non-collaborative" in `notes:`).
4. The plan body (shared context).
5. The agent's task paragraph (from recipe's `# Single-agent spec`).
6. Owned files / glob list.
7. Worktree path + branch when isolation is on.
8. Definition of Done (single variant): tests pass + Conventional Commits commit + one-paragraph final reply summarizing the change.

### Mode=single PR-flow

- **Reviewer-banner injection is unavailable.** No reviewer role exists in single mode.
- **Static disclosure notice is the only mode=single PR addition.** When `pr_create: true` AND `skip_qa: true` are both explicitly user-confirmed, run the same noninteractive `gh pr create` flow as mode=team but prepend this exact blockquote to `--body-file`:

  ```
  > **assemble-team mode=single**: this PR was produced by a single subagent.
  > No reviewer verdict and no QA verdict were collected.
  > The human PR reviewer is the only review gate.
  ```

- Without `skip_qa: true`, block PR creation. Report the missing flag. No silent retry.

### Mode=single output

Subagent's final response normalized into the report format with `verdicts: null`.

## Mode=manual execution

`execute-work` synthesizes the spawn prompt(s) and **prints** them. No `TeamCreate`, no `Agent` call.

For multi-card recipes (rare in manual — verify-mapping should have collapsed by then; only happens when user explicitly chose manual despite team-shaped mapping), print each teammate's prompt in its own fenced block, labeled by teammate name.

Automation flags are ignored (manual = user does it themselves).

Report's `result: printed`.

## Known limits (preserved from v0.1.0)

- `/resume` does not restore in-process teammates. The lead may try to message a teammate that no longer exists — respawn.
- Teammate task-status lag — the lead may need to refresh manually.
- Shutdown waits for the current turn to finish.
- One team per session.
- Nested teams are not supported.
- The lead role is bound to the session that created the team.

## Safety defaults

| Item | Default |
|---|---|
| Execution location | Current session (current window splits into panes) |
| User confirm before spawn | ON. Skip is not allowed. (Enforced upstream by verify-mapping.) |
| Teammate model | `sonnet` |
| Team size | 3-5 based on plan signals (mode=team) |
| Worktree isolation | OFF (ON when plan signals destructive ops or overlap) |
| Plan-approval mode | ON for risky tasks |
| Permission | Inherited from lead. Reviewer and read-only roles are adjusted post-spawn. |

## When NOT to use

- The plan has not been mapped to a recipe (use `verify-mapping` first).
- Single task or strongly sequential work (use a single subagent invocation directly).
- "Just give me an answer" research (use a single subagent).
````

- [ ] **Step 2: Verify the file**

```bash
head -25 plugins/assemble-team/skills/execute-work/SKILL.md
grep -c "^---$" plugins/assemble-team/skills/execute-work/SKILL.md
```

Expected: frontmatter starts with `---`, contains `name: execute-work`. Count of `---` lines `>= 2` (frontmatter open/close).

- [ ] **Step 3: Commit**

```bash
git add plugins/assemble-team/skills/execute-work/SKILL.md
git commit -m "feat(assemble-team): add execute-work sub-skill SKILL.md (L5 with mode branching + idempotency)"
```

### Task 8: Write `skills/assemble-team/SKILL.md` (orchestrator)

**Files:**
- Create: `plugins/assemble-team/skills/assemble-team/SKILL.md`
- Reference: spec §"Orchestrator: `assemble-team`" (lines ~322–345)

- [ ] **Step 1: Write the file**

Create `plugins/assemble-team/skills/assemble-team/SKILL.md` with this content:

````markdown
---
name: assemble-team
description: |
  Orchestrator skill that chains the three assemble-team sub-skills:
  enrich-plan → verify-mapping → execute-work. Each handoff is validated
  by checking the next skill's required frontmatter fields; on validation
  failure or user abort, halt and preserve partial output so the user can
  resume by invoking the next sub-skill manually.

  Trigger when the user says:
  - "/assemble-team ..." (slash command, full end-to-end chain)
  - "이 plan으로 팀 만들어줘" / "팀 에이전트로 돌려" / "team agents on this"
  - "assemble a team for this plan"
  - pastes a plan body + "process this in parallel" / "이거 병렬로 처리해"

  Do NOT use this skill when:
  - Single-task work (use a normal subagent)
  - Strongly sequential work (no parallel benefit)
  - Multiple workers would edit the same file (conflict)
---

# assemble-team (orchestrator)

The orchestrator skill chains three independently-invokable sub-skills:

```
raw plan
  → enrich-plan        ──→  enriched plan (frontmatter w/ ambiguity, classification, sources, forced_blocks_resolved)
  → verify-mapping     ──→  execution recipe (frontmatter w/ recipe_id, mode, automation, guards_version, guards_hash)
  → execute-work       ──→  execution report (frontmatter w/ recipe_id, result, pr_url, verdicts, notes)
```

## What this skill does

Sequence the three sub-skills with handoff validation. No mapping or execution business logic lives here — those belong to the sub-skills. The orchestrator owns:

1. Initial input intake (path / URL / inline)
2. Sub-skill invocation via the `Skill` tool
3. Frontmatter validation at each handoff
4. Halt-and-preserve on sub-skill failure or user abort
5. Returning the final execution report to the user

## Behavior

1. Receive initial input (raw plan).
2. Invoke `enrich-plan` skill via the `Skill` tool.
3. Validate the enriched plan output:
   - Frontmatter present
   - `forced_blocks_resolved: true`
   - `guesses_to_confirm: []` (empty — all guesses must be confirmed)
4. Invoke `verify-mapping` skill, passing the enriched plan.
5. Validate the recipe output:
   - `recipe_id` present
   - `mode` present and valid
   - `guards_version` and `guards_hash` present
6. Invoke `execute-work` skill, passing the recipe.
7. Return the final execution report to the user.

## Failure handling

On any sub-skill failure or user abort:

- Halt the chain immediately.
- Present the partial output (the last successful sub-skill's output) to the user.
- Note the next sub-skill to resume from. E.g., "enrich-plan completed; verify-mapping aborted. Re-run by invoking /verify-mapping with the enriched plan above when ready."
- The user can pass the partial output back into a downstream sub-skill manually to continue (e.g., `/verify-mapping "<paste enriched plan>"`).

## When NOT to use

- Single-task work (use a normal subagent).
- Strongly sequential work (no parallel benefit).
- Multiple workers would edit the same file (conflict).
- You only need step 1, 2, or 3 in isolation — invoke that sub-skill's slash command directly (`/enrich-plan`, `/verify-mapping`, `/execute-work`).

## Sub-skill quick reference

| Sub-skill | Layers | Input | Output |
|---|---|---|---|
| `enrich-plan` | L1+L2+L3 | raw plan (path/URL/inline) | enriched plan markdown |
| `verify-mapping` | L4 + mode selection | enriched plan markdown | execution recipe markdown |
| `execute-work` | L5 (mode-branched) | execution recipe markdown | execution report markdown |
````

- [ ] **Step 2: Verify the file**

```bash
head -25 plugins/assemble-team/skills/assemble-team/SKILL.md
```

Expected: frontmatter with `name: assemble-team` and the description.

- [ ] **Step 3: Commit**

```bash
git add plugins/assemble-team/skills/assemble-team/SKILL.md
git commit -m "feat(assemble-team): add assemble-team orchestrator SKILL.md (thin, sequence + validate)"
```

---

## Phase 4 — Slash command shims

Four command files, one per skill. Each is a thin shim that invokes its corresponding skill via the `Skill` tool. The frontmatter uses Claude Code's standard fields: `description`, `argument-hint`, `allowed-tools`.

### Task 9: Write `commands/enrich-plan.md`

**Files:**
- Create: `plugins/assemble-team/commands/enrich-plan.md`

- [ ] **Step 1: Write the file**

Create `plugins/assemble-team/commands/enrich-plan.md` with this exact content:

````markdown
---
description: Enrich a plan with classification + ambiguity scoring + grilling (step 1 of assemble-team)
argument-hint: <plan body | path to plan file | GitHub issue/PR URL>
allowed-tools: Skill, Read, Glob, Grep, Bash, AskUserQuestion, WebFetch
---

Invoke the `enrich-plan` skill from the assemble-team plugin on the plan body, path, or URL provided in the arguments. Pass the user input through verbatim:

$ARGUMENTS
````

- [ ] **Step 2: Verify**

```bash
head -10 plugins/assemble-team/commands/enrich-plan.md
```

Expected: frontmatter has `description`, `argument-hint`, and `allowed-tools` lines. Body references `$ARGUMENTS`.

- [ ] **Step 3: Commit**

```bash
git add plugins/assemble-team/commands/enrich-plan.md
git commit -m "feat(assemble-team): add /enrich-plan slash command shim"
```

### Task 10: Write `commands/verify-mapping.md`

**Files:**
- Create: `plugins/assemble-team/commands/verify-mapping.md`

- [ ] **Step 1: Write the file**

Create `plugins/assemble-team/commands/verify-mapping.md` with this exact content:

````markdown
---
description: Derive team cards + select execution mode (step 2 of assemble-team)
argument-hint: <enriched plan markdown | path to enriched plan file>
allowed-tools: Skill, Read, Glob, Grep, AskUserQuestion, Bash
---

Invoke the `verify-mapping` skill from the assemble-team plugin on the enriched plan provided in the arguments. The input must be an enriched plan (frontmatter contains `forced_blocks_resolved: true`):

$ARGUMENTS
````

- [ ] **Step 2: Verify**

```bash
head -10 plugins/assemble-team/commands/verify-mapping.md
```

Expected: frontmatter complete.

- [ ] **Step 3: Commit**

```bash
git add plugins/assemble-team/commands/verify-mapping.md
git commit -m "feat(assemble-team): add /verify-mapping slash command shim"
```

### Task 11: Write `commands/execute-work.md`

**Files:**
- Create: `plugins/assemble-team/commands/execute-work.md`

- [ ] **Step 1: Write the file**

Create `plugins/assemble-team/commands/execute-work.md` with this exact content:

````markdown
---
description: Execute an approved recipe in team / single / manual mode (step 3 of assemble-team)
argument-hint: <execution recipe markdown | path to recipe file>
allowed-tools: Skill, Read, Glob, Grep, Bash, AskUserQuestion, Agent, TeamCreate, TeamDelete, TaskList, SendMessage
---

Invoke the `execute-work` skill from the assemble-team plugin on the execution recipe provided in the arguments. The input must be a recipe (frontmatter contains `recipe_id` and `mode`):

$ARGUMENTS
````

- [ ] **Step 2: Verify**

```bash
head -10 plugins/assemble-team/commands/execute-work.md
```

Expected: frontmatter complete; `allowed-tools` includes `TaskList` for resume semantics.

- [ ] **Step 3: Commit**

```bash
git add plugins/assemble-team/commands/execute-work.md
git commit -m "feat(assemble-team): add /execute-work slash command shim"
```

### Task 12: Write `commands/assemble-team.md` (orchestrator)

**Files:**
- Create: `plugins/assemble-team/commands/assemble-team.md`

- [ ] **Step 1: Write the file**

Create `plugins/assemble-team/commands/assemble-team.md` with this exact content:

````markdown
---
description: Run the full enrich → map → execute chain (orchestrator entry point)
argument-hint: <plan body | path to plan file | GitHub issue/PR URL>
allowed-tools: Skill, Read, Glob, Grep, Bash, AskUserQuestion, WebFetch, Agent, TeamCreate, TeamDelete, TaskList, SendMessage
---

Invoke the `assemble-team` orchestrator skill on the plan body, path, or URL provided in the arguments. The orchestrator chains enrich-plan → verify-mapping → execute-work with validation at each handoff:

$ARGUMENTS
````

- [ ] **Step 2: Verify**

```bash
head -10 plugins/assemble-team/commands/assemble-team.md
```

Expected: frontmatter complete; `allowed-tools` is the union of all downstream tools.

- [ ] **Step 3: Commit**

```bash
git add plugins/assemble-team/commands/assemble-team.md
git commit -m "feat(assemble-team): add /assemble-team orchestrator slash command shim"
```

---

## Phase 5 — Documentation & marketplace integration

### Task 13: Write `plugins/assemble-team/README.md` (plugin-level)

**Files:**
- Create: `plugins/assemble-team/README.md`

- [ ] **Step 1: Write the file**

Create `plugins/assemble-team/README.md` with this content:

````markdown
# assemble-team

A Claude Code plugin that converts a user-provided plan into a safely-spawned execution. Three sub-skills (Enrich → Map → Execute) are callable independently or chained via `/assemble-team`. Universal — works with any monorepo.

## Install

```
/plugin marketplace add skarl86/claude-plugins
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
````

- [ ] **Step 2: Verify**

```bash
head -10 plugins/assemble-team/README.md
```

Expected: starts with `# assemble-team`.

- [ ] **Step 3: Commit**

```bash
git add plugins/assemble-team/README.md
git commit -m "feat(assemble-team): add plugin-level README for v0.2.0"
```

### Task 14: Update monorepo `marketplace.json`

**Files:**
- Modify: `.claude-plugin/marketplace.json`

- [ ] **Step 1: Read current marketplace.json**

```bash
cat .claude-plugin/marketplace.json
```

Expected: shows 4 existing plugins (github-direnv, claude-session-to-md, blog-illustrate, ralph-bootstrap) in a `plugins` array.

- [ ] **Step 2: Append assemble-team entry**

Insert this entry as the LAST element of the `plugins` array (before the closing `]`), preserving JSON validity (add a `,` to the previous element):

```json
{
  "name": "assemble-team",
  "source": "./plugins/assemble-team",
  "description": "5-layer harness (Enrich → Map → Execute) that converts a plan into a safely-spawned agent team, a single subagent, or a printed prompt. Three sub-skills callable independently or chained via /assemble-team."
}
```

The resulting file looks like:

```json
{
  "name": "skarl86-claude-plugins",
  "owner": {
    "name": "skarl86",
    "url": "https://github.com/skarl86"
  },
  "plugins": [
    {
      "name": "github-direnv",
      "source": "./plugins/github-direnv",
      "description": "Folder-scoped GitHub authentication via direnv (auto-switch gh / git push by directory)."
    },
    {
      "name": "claude-session-to-md",
      "source": "./plugins/claude-session-to-md",
      "description": "Convert Claude Code session jsonl logs into per-session markdown files for archive/search."
    },
    {
      "name": "blog-illustrate",
      "source": "./plugins/blog-illustrate",
      "description": "Generate clean illustrations (terminal mockups, decision trees, comparison cards) for blog posts. HTML/CSS → Playwright PNG → blog MCP upload → body insertion."
    },
    {
      "name": "ralph-bootstrap",
      "source": "./plugins/ralph-bootstrap",
      "description": "Bootstrap a Ralph loop scaffold (specs/, TODO.md, PROMPT.md, decisions.md, progress.md) from a one-sentence goal — state files for running Claude in a fixed-prompt while-loop."
    },
    {
      "name": "assemble-team",
      "source": "./plugins/assemble-team",
      "description": "5-layer harness (Enrich → Map → Execute) that converts a plan into a safely-spawned agent team, a single subagent, or a printed prompt. Three sub-skills callable independently or chained via /assemble-team."
    }
  ]
}
```

- [ ] **Step 3: Validate JSON**

```bash
python3 -m json.tool .claude-plugin/marketplace.json > /dev/null && echo OK
```

Expected: `OK`.

- [ ] **Step 4: Commit**

```bash
git add .claude-plugin/marketplace.json
git commit -m "feat: list assemble-team v0.2.0 in monorepo marketplace.json"
```

### Task 15: Update monorepo `README.md`

**Files:**
- Modify: `README.md` (monorepo root)

- [ ] **Step 1: Read current README.md**

```bash
cat README.md
```

Confirm it has sections: install instructions + `## Plugins` listing 4 plugin entries + `## Layout` + `## License`.

- [ ] **Step 2: Add the install line for assemble-team**

Find the `Then install whichever plugin(s) you need:` block (around line 15) and add a new line for `assemble-team` to the code fence. The new block:

```
/plugin install github-direnv
/plugin install claude-session-to-md
/plugin install blog-illustrate
/plugin install ralph-bootstrap
/plugin install assemble-team
```

- [ ] **Step 3: Add the plugin section**

After the `### [ralph-bootstrap](plugins/ralph-bootstrap)` section (and before the `---` separator that precedes `More plugins will be added over time.`), insert this new section:

```markdown
### [assemble-team](plugins/assemble-team)

Convert a user plan into a safely-spawned execution via a 3-step sub-skill chain (Enrich → Map → Execute) with mode branching (team / single / manual). Sub-skills are independently callable (`/enrich-plan`, `/verify-mapping`, `/execute-work`) and chained by `/assemble-team`. The 5-layer harness (intent classification, ambiguity scoring + grilling, role mapping, user approval, handoff with optional PR-flow) survives the split; mode branching lets the user run without a full team when one writer suffices.

**Use when:** a plan covers parallelizable work across 3+ scopes, or you want the option to escalate from "single subagent" to "team" without rewriting the plan.

State directory at `~/.claude/plugins/assemble-team/state/{started,completed,aborted}/` provides idempotency: re-running with the same `recipe_id` after completion is a safe no-op.
```

- [ ] **Step 4: Verify**

```bash
grep -A 2 "assemble-team" README.md | head -20
```

Expected: both the install line and the new section visible.

- [ ] **Step 5: Commit**

```bash
git add README.md
git commit -m "docs: announce assemble-team v0.2.0 in monorepo README"
```

---

## Phase 6 — Manual verification

This phase walks through a subset of the 29 test scenarios in the spec (sufficient for v0.2.0 acceptance). Each scenario is a manual check, not an automated test.

### Task 16: Migration integrity test — empty diffs

**Test scenario:** spec D.11 (verbatim moves)

- [ ] **Step 1: Run the three diff checks**

```bash
diff -q ~/.claude/plugins/marketplaces/skarl86-claude-plugins/plugins/assemble-team/skills/assemble-team/ROLES.md \
        plugins/assemble-team/skills/verify-mapping/ROLES.md
diff -q ~/.claude/plugins/marketplaces/skarl86-claude-plugins/plugins/assemble-team/skills/assemble-team/GRILL_PLAN.md \
        plugins/assemble-team/skills/enrich-plan/GRILL_PLAN.md
diff -q ~/.claude/plugins/marketplaces/skarl86-claude-plugins/plugins/assemble-team/skills/assemble-team/PLAN_TEMPLATE.md \
        plugins/assemble-team/skills/enrich-plan/PLAN_TEMPLATE.md
```

Expected: no output from any of the three (empty diff = success). If any diff shows content, fail and re-do the corresponding Task 2/3/4.

- [ ] **Step 2: Verify plugin layout matches the spec**

```bash
find plugins/assemble-team -maxdepth 4 -type f | sort
```

Expected list (13 files):

```
plugins/assemble-team/.claude-plugin/plugin.json
plugins/assemble-team/README.md
plugins/assemble-team/commands/assemble-team.md
plugins/assemble-team/commands/enrich-plan.md
plugins/assemble-team/commands/execute-work.md
plugins/assemble-team/commands/verify-mapping.md
plugins/assemble-team/skills/assemble-team/SKILL.md
plugins/assemble-team/skills/enrich-plan/GRILL_PLAN.md
plugins/assemble-team/skills/enrich-plan/PLAN_TEMPLATE.md
plugins/assemble-team/skills/enrich-plan/SKILL.md
plugins/assemble-team/skills/execute-work/SKILL.md
plugins/assemble-team/skills/verify-mapping/ROLES.md
plugins/assemble-team/skills/verify-mapping/SKILL.md
```

- [ ] **Step 3: No commit (verification only)** — proceed to next task.

### Task 17: Code-review invariant — no orphan-lock patterns in execute-work

**Test scenario:** spec F.27b (code-review invariant)

- [ ] **Step 1: Grep for forbidden patterns**

```bash
grep -nE "mkdir.*lock|\.lock/" plugins/assemble-team/skills/execute-work/SKILL.md
```

Expected: zero matches (no output). The execute-work SKILL.md should never instruct the executor to create a `.lock` directory; sentinels are always flat files.

If matches are found: edit the SKILL.md to remove them, then commit `fix(assemble-team): remove orphan-lock pattern from execute-work`.

### Task 18: JSON validity — manifest + marketplace

**Test scenario:** internal consistency

- [ ] **Step 1: Validate both JSON files**

```bash
python3 -m json.tool plugins/assemble-team/.claude-plugin/plugin.json > /dev/null && echo "plugin.json OK"
python3 -m json.tool .claude-plugin/marketplace.json > /dev/null && echo "marketplace.json OK"
```

Expected:

```
plugin.json OK
marketplace.json OK
```

- [ ] **Step 2: Verify the assemble-team entry is the 5th plugin in marketplace.json**

```bash
python3 -c "import json; m = json.load(open('.claude-plugin/marketplace.json')); names = [p['name'] for p in m['plugins']]; print(names); assert names[-1] == 'assemble-team', f'last plugin is {names[-1]}'"
```

Expected: a list of 5 plugin names ending with `'assemble-team'` and no AssertionError.

### Task 19: Frontmatter validity — every skill file

**Test scenario:** every skill loads via the Skill tool, which requires valid frontmatter

- [ ] **Step 1: Check the frontmatter of all 4 skill files**

```bash
for f in plugins/assemble-team/skills/*/SKILL.md; do
  echo "=== $f ==="
  head -2 "$f"
  if ! head -1 "$f" | grep -q '^---$'; then
    echo "FAIL: $f does not start with ---"
    exit 1
  fi
done
```

Expected: all four files start with `---` on line 1. No FAIL output.

- [ ] **Step 2: Check the `name:` field matches the directory**

```bash
for d in assemble-team enrich-plan verify-mapping execute-work; do
  f="plugins/assemble-team/skills/$d/SKILL.md"
  expected_name="$d"
  actual_name=$(grep -m 1 "^name:" "$f" | awk '{print $2}')
  if [ "$actual_name" != "$expected_name" ]; then
    echo "FAIL: $f has name=$actual_name, expected $expected_name"
    exit 1
  fi
  echo "OK: $f -> name: $actual_name"
done
```

Expected:

```
OK: plugins/assemble-team/skills/assemble-team/SKILL.md -> name: assemble-team
OK: plugins/assemble-team/skills/enrich-plan/SKILL.md -> name: enrich-plan
OK: plugins/assemble-team/skills/verify-mapping/SKILL.md -> name: verify-mapping
OK: plugins/assemble-team/skills/execute-work/SKILL.md -> name: execute-work
```

### Task 20: Slash command frontmatter — $ARGUMENTS + allowed-tools sanity

**Test scenario:** every command file uses the Claude Code command frontmatter

- [ ] **Step 1: Check each command file**

```bash
for f in plugins/assemble-team/commands/*.md; do
  echo "=== $f ==="
  # Must contain $ARGUMENTS in the body
  if ! grep -q '\$ARGUMENTS' "$f"; then
    echo "FAIL: $f does not reference \$ARGUMENTS"
    exit 1
  fi
  # Must have description, argument-hint, allowed-tools in frontmatter
  for field in description argument-hint allowed-tools; do
    if ! head -10 "$f" | grep -q "^$field:"; then
      echo "FAIL: $f missing $field in frontmatter"
      exit 1
    fi
  done
  # Skill must be in allowed-tools
  if ! head -10 "$f" | grep -q "^allowed-tools:.*Skill"; then
    echo "FAIL: $f allowed-tools missing Skill"
    exit 1
  fi
  echo "OK"
done
```

Expected: all four pass with `OK`.

- [ ] **Step 2: Verify TaskList is in execute-work and assemble-team commands**

```bash
for f in plugins/assemble-team/commands/execute-work.md plugins/assemble-team/commands/assemble-team.md; do
  if ! head -10 "$f" | grep -q "^allowed-tools:.*TaskList"; then
    echo "FAIL: $f allowed-tools missing TaskList (needed for resume semantics)"
    exit 1
  fi
  echo "OK: $f has TaskList"
done
```

Expected:

```
OK: plugins/assemble-team/commands/execute-work.md has TaskList
OK: plugins/assemble-team/commands/assemble-team.md has TaskList
```

### Task 21: Guards hash compute — spec consistency check

**Test scenario:** the guards_hash value defined in verify-mapping/SKILL.md is reproducible

- [ ] **Step 1: Compute the canonical guards_hash**

```bash
guards_hash=$(printf "%s\n" "Boolean defaults" "Branching" "CI / deploy files" "Commit convention" "No auto-push / no auto-merge" "Package manager respect" "Secrets" "Teammate spawn guard" | sort | sha256sum | cut -c1-12)
echo "guards_hash = $guards_hash"
```

Expected: a 12-character hex string. Record the value — it will be the canonical value verify-mapping should emit. (No spec test failure here — this is a sanity step. The spec does NOT pin a specific hash; the hash is computed at runtime by verify-mapping using the same shell snippet.)

### Task 22: Final commit & PR-ready status

**Test scenario:** branch is ready for PR

- [ ] **Step 1: Status check**

```bash
git status
```

Expected: `nothing to commit, working tree clean`. All previous tasks committed.

- [ ] **Step 2: Log summary**

```bash
git log --oneline main..HEAD
```

Expected: a sequence of feat/docs commits (one per task in Phases 1–5, plus the spec history from before). The most recent ~15 should match the task order: plugin.json → 3 verbatim copies → 4 SKILL.md → 4 command shims → plugin README → marketplace.json → monorepo README.

- [ ] **Step 3: Confirm all files exist and are tracked**

```bash
git ls-files plugins/assemble-team | wc -l
```

Expected: `13` total files (1 plugin.json + 1 plugin README + 4 commands + 4 SKILL.md + 3 reference docs). If less, re-run the file listing in Task 16 to find what's missing.

- [ ] **Step 4: No code commit here — only verification.**

The branch `worktree-assemble-team-v2-design` is now implementation-complete and ready for PR. Handoff to the user happens at the end of the plan execution (see "After this plan" below).

---

## After this plan

Once all tasks in this plan are complete:

1. **Manual smoke test** (recommended before PR): run `/assemble-team` against the refund-dialog example in `plugins/assemble-team/skills/enrich-plan/PLAN_TEMPLATE.md`. Expected: same end-to-end behavior as v0.1.0 (team mapping → user approval → TeamCreate → PR-flow). If this regresses, fix and commit before opening the PR.
2. **PR creation**: `gh pr create --draft --base main --title "Add assemble-team v0.2.0 plugin (migrate + split + mode branching)" --body-file <(cat <<EOF
## Summary
- Migrate \`assemble-team\` plugin into this monorepo at \`plugins/assemble-team/\`.
- Split the single v0.1.0 skill into 3 sub-skills (\`enrich-plan\` / \`verify-mapping\` / \`execute-work\`) plus a thin orchestrator (\`assemble-team\`).
- Add mode branching: \`team\` (current behavior), \`single\` (one subagent), \`manual\` (print prompts).
- Add idempotency via per-recipe state files in \`~/.claude/plugins/assemble-team/state/\`.
- Spec: \`docs/superpowers/specs/2026-05-19-assemble-team-v2-design.md\` (785 lines, 6-round Codex-reviewed).
- Implementation plan: \`docs/superpowers/plans/2026-05-19-assemble-team-v0.2.0.md\`.

## Test plan
- [ ] All Task 16–22 verification steps pass.
- [ ] Smoke test against PLAN_TEMPLATE.md refund-dialog example matches v0.1.0 happy-path behavior.

🤖 Generated with [Claude Code](https://claude.com/claude-code)
EOF
)`

---

## Self-review checklist (run before declaring done)

- [ ] Every task has bite-sized steps (2–5 minutes each).
- [ ] Every step shows the exact command or file content; no "TBD" / "fill in" / "similar to Task N".
- [ ] File paths are exact and consistent across tasks.
- [ ] Frontmatter naming is consistent (canonical snake_case in skill outputs; aliases documented).
- [ ] All 4 SKILL.md files reference the same `guards_version` and the same canonical guard list.
- [ ] Idempotency contract uses flat-file sentinels with `set -C` (noclobber) and per-PID temp filenames — no `mkdir`-based locks, no shared temp paths.
- [ ] The plan's task ordering matches the spec's "Migration order" section.
- [ ] Phase 6 verification scenarios cover: empty-diff migrations (16), no-orphan-lock invariant (17), JSON validity (18), frontmatter validity (19), command shim validity (20), guards_hash reproducibility (21), final status (22).
