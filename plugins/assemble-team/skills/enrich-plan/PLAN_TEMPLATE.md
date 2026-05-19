# Plan template (assemble-team friendly)

Writing your plan in this shape lets the lead derive a team accurately. Of the sections below, **Why / Scope / Tasks / DoD / Out of scope** are recommended for any non-trivial plan.

## Recommended sections

### Why (required)
One paragraph: why this work is needed, what triggered it, and any prior context. The lead uses this to score the Goal dimension and to write commit / PR messages.

### Scope (strongly recommended)
List the directories / services that this plan touches. One line per scope. Examples:
- `apps/<app-name>` (frontend)
- `services/<service-name>` (backend)
- `packages/<shared-lib>`
- `infra/`, `.github/workflows/` (devops scope)

The lead maps these to roles using `ROLES.md`.

### Constraints (close to required)
- read-only / write
- token / time budget
- explicit do-not-touch items (e.g. no push, no infra changes)
- worktree isolation (`worktree=on` / `worktree=off`)
- permissions (`permissions=skip` / `permissions=ask` / `permissions=accept-edits`)
- automation flags (optional):
  - `automation: pr-create` — when all qa tasks are `✅` and worktree isolation is OFF, the lead runs `gh pr create --draft --base <default-branch>`. Merge is never automatic.
  - `automation: skip-qa` — suppress the default-added qa task (use for read-only investigative plans).
  - `automation: allow-push` — pre-authorize the lead to run `git push -u origin <branch>` before `gh pr create`. Without this flag the lead asks once at PR time and proceeds only on explicit yes.

### Tasks (required)
One line per task. Optional `[role:X]` hint at the end:
- `- [ ] [role:frontend] update the RefundDialog copy`
- `- [ ] [role:backend] add a new field to the refund response schema`
- `- [ ] [role:reviewer] cross-review the changes above`

If `[role:X]` is omitted, the lead infers the role from `ROLES.md` heuristics.

### Definition of Done (required)
Per task, a verifiable outcome:
- changed files committed (Conventional Commits)
- relevant tests pass (`pytest` / `vitest` / etc.)
- one-paragraph summary `SendMessage` to lead
- (optional) post-deploy / runtime verification

### Out of scope (strongly recommended)
Items deliberately deferred to a human or later iteration:
- PR creation / push / merge / deploy
- adjacent refactors at the same call sites
- dependency upgrades
- external system changes (schema, infra env vars)

## Optional sections

### Risks
- regression risk (e.g. "external API schema change → parser mismatch")
- data risk (e.g. "DB migration with no easy rollback")

### Token budget
- estimated tokens overall and per teammate

### Time budget
- target wall-clock

---

## Example plan (universal — single writer)

This example is intentionally a **single-writer** plan so that `worktree=off` is unambiguously correct and the `automation: pr-create` flow is end-to-end demonstrable. For plans with two or more code-modifying teammates, follow `GRILL_PLAN.md` step 4: `worktree=on` is required (and `automation: pr-create` then becomes unavailable until the user merges the branches manually).

```markdown
# Plan: refund-dialog copy refresh

## Why
Q2 introduces a new refund policy. The UI copy on the RefundDialog must reflect
the policy change. Inconsistent wording has been flagged by support as a
recurring complaint source. The backend response schema and the messaging
templates have already been updated; this plan covers the UI surface only.

## Scope
- apps/web/src/components/RefundDialog.tsx (the only code-modifying scope)

## Constraints
- worktree=off (single code-modifying teammate; no Git index race)
- permissions=skip
- Conventional Commits
- push / merge / deploy → human only
- automation: pr-create  (lead creates a draft PR when qa is `✅` AND worktree isolation is OFF; ready and merge remain human)

## Tasks
- [ ] [role:frontend] update RefundDialog copy to match the new policy
- [ ] [role:reviewer] cross-review the change for wording, secrets, boolean defaults
- [ ] [role:qa] (default-added) component test + RefundDialog e2e on top of the per-surface self-check — the verdict gates the automation: pr-create flow

## Definition of Done
- The frontend change is committed on a branch from the default branch (Conventional Commits)
- `pnpm test` passes in `apps/web`
- reviewer: `✅` / `⚠️` / `❌` table delivered
- qa: final verdict `all ✅` (or the project's locale equivalent) — gates the PR auto-creation
- automation: pr-create runs → draft PR opened; ready and merge remain human

## Out of scope
- payment gateway changes
- backend refund response schema (already updated in a previous plan)
- messaging templates (already updated)
- promoting the draft PR to ready or merging it (human only)

## Risks
- wording drift between UI and the already-shipped backend message — reviewer must catch
- a single qa `❌` blocks the automation: pr-create flow — fix and rerun
```

When you invoke the assemble-team skill against this body (paste the plan and ask "assemble a team for this plan"), the lead will:
1. Map the one scope — `RefundDialog.tsx` → `frontend`
2. Add `[role:reviewer]` and the default `[role:qa]` to reach a 3-teammate team (frontend writer + 2 read-only roles)
3. Show the team mapping + ambiguity summary + `automation: pr-create` notice for approval
4. Spawn with `worktree=off` (only one code-modifying teammate; no Git index race), `permissions=skip`, role bodies inline-injected
5. After all teammates idle, check the qa verdict. If `✅` AND worktree isolation is OFF (which it is in this example), the lead:
   a. discovers the default branch — `DEFAULT_BRANCH=$(gh repo view --json defaultBranchRef --jq .defaultBranchRef.name)`;
   b. reconciles the no-auto-push guard — if `automation: pr-create` is set without `automation: allow-push` AND no explicit grilling consent was recorded, the lead asks the user once "Push the branch and create the draft PR? (yes / no)" and proceeds only on yes; otherwise it pushes via `git push -u origin "$(git rev-parse --abbrev-ref HEAD)"`;
   c. synthesizes `PR_TITLE` (one line, ≤ 70 chars, derived from the plan's `Why`) and `PR_BODY_FILE` (`$(mktemp)` containing the full body — plan `Why` + per-teammate report + qa verdict + reviewer summary banner when applicable);
   d. runs the noninteractive command exactly as L5 instructs:

```bash
gh pr create --draft \
  --base "$DEFAULT_BRANCH" \
  --head "$(git rev-parse --abbrev-ref HEAD)" \
  --title "$PR_TITLE" \
  --body-file "$PR_BODY_FILE"
```

Both `--title` and `--body-file` MUST be present so `gh` never falls into an interactive editor and hangs the agent run.

If qa is not green or any precondition fails, the lead reports the failure to the user and stops without creating a PR.
