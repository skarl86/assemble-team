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

## Parsing rules (input handling)

These three rules apply to the input the sub-skill receives, before any field-level validation:

1. **Code-fence stripping**: when the input is wrapped in a markdown code fence (e.g., ```` ```markdown\n...\n``` ````), strip the outer fence before extracting the YAML frontmatter. This is a common shape when a user pastes an LLM-formatted block. Log the strip in the output's `notes:` list (e.g., "outer code fence stripped").
2. **Duplicate-frontmatter rejection**: two consecutive `---`...`---` blocks at the very top of the input (separated only by blank lines) → fatal: "Multiple frontmatter blocks detected; remove the extra block." A `---` line that appears inside the body section (e.g., as a markdown horizontal rule after non-trivial content) is NOT a frontmatter block and is ignored by this rule.
3. **Raw-input preservation in fatal errors**: every fatal validation error must include the user's raw input verbatim in the message, so the user can edit and retry without re-typing.

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
