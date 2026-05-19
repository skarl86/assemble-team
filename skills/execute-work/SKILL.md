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

## Parsing rules (input handling)

These three rules apply to the input the sub-skill receives, before any field-level validation:

1. **Code-fence stripping**: when the input is wrapped in a markdown code fence (e.g., ```` ```markdown\n...\n``` ````), strip the outer fence before extracting the YAML frontmatter. This is a common shape when a user pastes an LLM-formatted block. Log the strip in the output's `notes:` list (e.g., "outer code fence stripped").
2. **Duplicate-frontmatter rejection**: two consecutive `---`...`---` blocks at the very top of the input (separated only by blank lines) → fatal: "Multiple frontmatter blocks detected; remove the extra block." A `---` line that appears inside the body section (e.g., as a markdown horizontal rule after non-trivial content) is NOT a frontmatter block and is ignored by this rule.
3. **Raw-input preservation in fatal errors**: every fatal validation error must include the user's raw input verbatim in the message, so the user can edit and retry without re-typing.

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
