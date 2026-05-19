# GRILL_PLAN — turn a thin plan into a complete plan

The lead enters this procedure when L3 Enrichment determines the plan is incomplete. The goal is to fill missing fields one question at a time, in dependency order, while preferring codebase exploration to user interrogation.

## Core principles

1. **One question at a time** — bundling 5 questions into one round breaks the user.
2. **Every question carries a recommendation** — the user only confirms or adjusts.
3. **Follow dependency order** — Scope must be set before Tasks make sense.
4. **Prefer code search over asking** — use `Glob` / `Read` / `Grep` first; ask only when discovery fails.
5. **Source-tag every filled field** — `[from-user]` / `[from-code: path:line]` / `[guess]`. `[guess]` flags must be confirmed before L4 approval.

## Ambiguity score (quantitative gate)

Score 5 dimensions on a 0.0-1.0 scale based on the plan body. No LLM calls — natural-language judgment.

| Dimension | 0.0 (missing / vague) | 1.0 (fully specified) |
|---|---|---|
| **Goal (Why)** | one-line label like "improve refunds" | one-paragraph why, with incident / KPI / motivation |
| **Scope** | no directory hints | concrete paths or globs (e.g. `services/payments/`) |
| **Tasks** | 0 tasks, or "clean it up" | action-verb + concrete file/symbol per task |
| **Constraints** | absent + destructive signals present | explicit constraints ("no DB migration", "branch from main") |
| **DoD** | absent or "works well" | verifiable outcome (`pytest passes`, `e2e: refund=true flow`) |

Aggregation:

```
Clarity   = (Goal + Scope + Tasks + Constraints + DoD) / 5
Ambiguity = 1 - Clarity
```

| Ambiguity | Handling |
|---|---|
| ≤ 0.30 | Pass — skip grilling, go to L4. |
| 0.30 < x ≤ 0.50 | Grill the lowest-scoring dimensions, but always ask in dependency order (Why → Scope → ...). |
| > 0.50 | Force-enter grilling. Even on user impatience, secure at least Goal + Tasks. |

**Threshold rationale.** omc's deep-interview uses 0.20. This skill uses 0.30 to be more permissive of short fix-it plans. Forced-block heuristics below act as a second safety net.

**Forced-block heuristics (binary — block even if the score passes)**:

- `Goal` missing
- `Tasks` count == 0
- `Constraints` missing AND destructive signal present (DB migration, drop / truncate, force-push, infra deletion)

## Grilling dependency order (7 steps)

Do not skip ahead. Earlier answers shape later recommendations.

```
1. Why          (intent)
   ↓
2. Scope        (which directory / service)
   ↓
3. Tasks        (what to do in each scope)
   ↓
4. Constraints  (read-only? worktree? permissions? automation?)
   ↓
5. DoD          (per-task completion criteria)
   ↓
6. Out of scope (explicit exclusions — safety net)
   ↓
7. Risks        (regression risk derived from scope + tasks + domain)
```

For each step, the lead's flow is:

```
(1) Is this field already in the plan?
    ├─ yes → "I read this as ⟨X⟩, correct?" — one-line confirmation; skip on user yes.
    └─ no  → (2)
(2) Can the lead discover the answer from the codebase?
    ├─ yes → search, present finding + "I inferred ⟨X⟩, correct?"
    └─ no  → (3)
(3) `AskUserQuestion` — one question, with one recommended answer (or 2-3 options).
(4) Record the user's answer in plan memo with source tag → go to next step.
```

## Per-step recommendations (generic — no domain assumptions)

### 1. Why

| Signal in the plan body | Recommended Why |
|---|---|
| "bug", "user complaint", "regression" | "Block a user-facing regression quickly." |
| "policy change", "terms update" | "Maintain consistency with the new policy." |
| "new feature", "launch", "MVP" | "Prepare for the new feature launch." |
| No signal | **Ask the user — no skip allowed.** |

Sample question: "Why is this work needed? Recommendation: ⟨lead's one-line inference⟩. Accept as-is or rewrite in one line."

### 2. Scope

First try codebase discovery:
- Are there directory keywords in the plan?
- If yes, `Glob` to confirm those paths exist → "I inferred Scope as ⟨paths⟩, correct?"

If discovery fails, ask with options:
- frontend (which app?)
- backend (which service?)
- shared library / package
- CI / deploy / infra
- multiple (multi-select)

Tagging examples:
- `[from-code: services/payments/refund.py:42 confirmed]` → `services/payments/`
- `[guess] services/` ⚠️ — flag for user confirmation
- `[from-user]` plan body literal

### 3. Tasks

If Tasks is sparse or 1, the lead proposes a decomposition:
- One task per Scope by default.
- One task for cross-review if multiple implementers exist.
- One `[role:qa]` task by default when any code-modifying task exists (suppress with `automation: skip-qa`).
- One task for research first when the plan signals investigation.

Sample decomposition prompt:
```
"Refund policy update" — proposed decomposition:
  - [role:frontend] update the RefundDialog message
  - [role:backend] add `refund_eligibility_reason` to the refund response schema
  - [role:qa] integration e2e on top of per-scope self-checks
  - [role:reviewer] cross-review the three changes
Accept, or add / remove items?
```

If the plan body contains `[role:X]` hints, respect them.

### 4. Constraints

Recommendation heuristics (the SKILL safety default is `worktree=off`, but any plan with two or more code-modifying teammates MUST run with `worktree=on` because they share the same Git index otherwise):
- "investigate", "audit", "analyze" → `read-only=on`, `worktree=off`
- exactly one code-modifying teammate (other teammates are read-only — reviewer / researcher / qa) → `worktree=off`, `permissions=skip`. The `automation: pr-create` flow works with this configuration because only one writer touches the index.
- two or more code-modifying teammates (regardless of whether their files overlap) → `worktree=on`, `permissions=skip`. Concurrent writers in a single checkout race on `git add` / `git commit` even with non-overlapping files; per-teammate worktrees prevent this. The `automation: pr-create` flow is unavailable until the user merges the branches manually.
- contains "production deploy" / "DB migration" / "force-push" → `worktree=on`, `permissions=ask`, `push=block`
- no signal → defaults (`worktree=off`, `permissions=skip`, `push=block`)

Alternative for two or more writers in a single checkout: the lead can serialize teammate commits explicitly (one teammate finishes and reports `done` before the next is allowed to commit). This requires extra orchestration and is only acceptable when the user opts out of `worktree=on`; otherwise prefer worktree isolation.

`automation:` flags (recognized literals — must be set by the user, never by lead inference of natural-language signals):

| Value | Effect |
|---|---|
| `automation: pr-create` | After all qa `✅` AND worktree isolation OFF, the lead runs the noninteractive `gh pr create --draft --title ... --body-file ...` flow. Merge is always human. |
| `automation: skip-qa` | Suppress the default-added qa task. Use when the work is read-only / investigative. |
| `automation: allow-push` | Pre-authorize the lead to run `git push -u origin <branch>` before `gh pr create`. Without this flag, the lead asks once at PR time and proceeds only on explicit yes. |
| Combinations | Any combination is allowed (e.g. `automation: pr-create, allow-push`). When `pr-create + skip-qa` are both set, the lead confirms once: "Create a PR without qa verification?" — proceed only on yes. |

**How the lead recognizes that a flag is "user-set"** (the L5 gate uses the same definition):
1. The flag literal (e.g. `automation: pr-create`) is present in the original plan body that triggered this skill — recorded as `[from-user]`. OR
2. The user gave an explicit yes to a grilling option for that flag — also recorded as `[from-user]` on the Constraints field. OR
3. The flag was added by an explicit user reply to a focused question during grilling ("Add `automation: pr-create`?" → yes).

Lead-inferred natural-language signals ("create a PR for me", "올려줘", "그대로 push 해") are NEVER sufficient. The lead must obtain an explicit yes for any automation flag and record it with `[from-user]`.

Suggest the defaults with a 4-option `AskUserQuestion`:
1. Recommended default (one-line summary including any `automation:` flags inherited from the plan body)
2. More conservative — `read-only=on`
3. `permissions=ask` for per-step approval
4. Add `automation: pr-create` (and optionally `automation: allow-push` if the user also wants the lead to push before opening the PR)

### 5. DoD

Per-task default DoD:
- code-modifying task → "tests pass + Conventional Commits commit + one-line report to lead via `SendMessage`"
- investigation task → "one-paragraph report + file path / line citations + 1-2 open questions"
- review task → "`✅` / `⚠️` / `❌` table with a one-line justification per change"

Ask: "Does this DoD work?"

### 6. Out of scope (safety net)

Default exclusions:
- PR creation / merge / push / deploy (these are human decisions)
- adjacent refactors not requested
- dependency upgrades
- external system changes (DB schema, infra env vars) unless plan explicitly includes them

Ask: "Anything else to explicitly exclude? (no answer → defaults apply)"

### 7. Risks (regression)

Generic risk catalog (always inject when relevant):
- External API schema change → parser / consumer mismatch
- DB migration → rollback may be difficult
- Deploy pipeline → missing env vars → boot failure
- Cross-surface wording change → consistency drift between UI / API / messaging
- `automation: pr-create` + any qa `❌` → PR auto-creation blocks; user must fix and rerun

Project-specific risks are out of scope for this skill — the user supplies them via the plan body or a project-level extension.

Ask: "These risks have been added: ⟨list⟩. Any missing?"

## Closing the grilling — complete plan summary

When the 7 steps are resolved, the lead shows the user a single summary block and asks for final confirmation before L4.

```
✅ Plan enriched.

Ambiguity score: 0.24  (threshold 0.30 — pass)
  Goal       : 0.9
  Scope      : 0.8
  Tasks      : 0.8
  Constraints: 0.7
  DoD        : 0.6

# Why
[from-user] ⟨filled Why⟩

# Scope
[from-code: ⟨path:line⟩] ⟨filled paths⟩
[from-user] ⟨user-supplied paths⟩

# Tasks
[from-user] ⟨filled task list⟩

# Constraints
[from-user] ⟨filled constraints⟩
[guess] ⚠️ ⟨lead-inferred constraint — confirm before proceeding⟩

# DoD
[from-user] ⟨filled DoD⟩

# Out of scope
[from-user] ⟨filled exclusions⟩

# Risks
[from-code] ⟨domain-derived risks with citations⟩

Proceed to L4 (team mapping)? yes / amend?
```

If any `[guess]` remains, the lead adds a one-line emphasis: "Confirm or replace?"

## Anti-patterns (do not do)

- ❌ Bundle 5-7 fields in one round — the user cannot answer.
- ❌ Ask "what would you like?" without a recommendation — the user has to think from scratch.
- ❌ Ask the user something the codebase can answer — wastes the user's time.
- ❌ Re-ask a field the user already answered — keep a grilling memo.
- ❌ Pose Risks generically ("any risks?") — use the catalog above.
- ❌ Extend grilling beyond 7 rounds — collapse into defaults.

## User-impatience signals → collapse grilling

If the user says "skip", "just do it", "default", "fine", "enough" — secure only forced-block items (Goal / Tasks / destructive signals) and proceed. Mark uncovered fields as defaults with a red flag in the L4 approval prompt.
