---
name: assemble
description: |
  Orchestrator skill of the assemble-team plugin. Chains the three sibling
  sub-skills: enrich-plan → verify-mapping → execute-work. Each handoff is
  validated by checking the next skill's required frontmatter fields; on
  validation failure or user abort, halt and preserve partial output so the
  user can resume by invoking the next sub-skill manually.

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

# assemble (orchestrator)

The orchestrator skill of the assemble-team plugin. Chains three independently-invokable sibling sub-skills:

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
