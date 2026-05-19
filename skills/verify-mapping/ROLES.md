# assemble-team — Role Catalog and Routing Rulebook

The lead reads this file in L2 Routing to (a) classify the plan's intent, and (b) map plan signals to teammate roles. In L5 Handoff the lead injects the chosen role bodies into spawn prompts.

## Intent classification signals (used by L2 Routing — Axis 1)

The lead picks one Intent for the plan. When multiple intents could apply, the lead picks the one that best characterizes the work *as a whole* and notes the secondary in the L4 approval prompt.

| Signal in the plan body or user phrasing | Intent |
|---|---|
| Single-file, < 10 LoC, obvious fix; phrases like "fix the typo", "rename", "bump version" | **Trivial** |
| `refactor`, `restructure`, `clean up`, `extract`, `rename module`; existing-code reshaping with no behavior change | **Refactor** |
| `create new`, `add feature`, `greenfield`, `scaffold`, new module / package / service | **Build** |
| Scoped feature with concrete deliverables (e.g. "add an API endpoint", "implement the onboarding flow") | **Mid-sized** |
| `let's figure out`, `help me plan`, `walk me through`, user clearly wants dialogue / iteration | **Collaborative** |
| `how should we structure`, `system design`, infrastructure / cross-service decisions, long-term shape | **Architecture** |
| Goal exists but path is unclear; "investigate", "audit", "how does X work" with no clear deliverable | **Research** |

When none of the signals match, the lead asks the user one targeted question: "Which best fits — ⟨top-2 intents⟩?" — and proceeds with the user's pick.

## Plan → Role heuristic (used by L2 Routing — role mapping)

| Signal in the plan body | Recommended role |
|---|---|
| `frontend/...`, `apps/web`, Next.js, React, Vue, Svelte, Tailwind, shadcn, Vitest, Playwright (UI portion) | **frontend** |
| `backend/...`, `services/...`, FastAPI, Express, NestJS, Django, Spring, GraphQL, REST handlers, DB migrations | **backend** |
| `.github/workflows/`, `Dockerfile`, `k8s/`, `terraform/`, Cloud Run, Vercel, env-var changes, CI/CD pipelines | **devops** (optional) |
| `test`, `spec`, `regression`, `e2e`, `vitest`, `pytest`, `jest`, `playwright` | **qa** |
| "investigate", "how does X work", "where is X called", "map the data flow" | **researcher** |
| "code review", "security review", "is this safe" | **reviewer** |
| "split this plan", "what's missing", dependency analysis | **architect** |
| "challenge this hypothesis", "what view is missing", competing-hypotheses debugging | **devils-advocate** (optional) |

When multiple signals fire, prefer one teammate per scope. Use the `-a` / `-b` suffix only when the same scope genuinely needs parallel workers. Keep team size in the 3-5 range.

---

## Role: frontend

- **Responsibilities**: UI of web (or mobile-web) applications. Componentry, state, styling, client-side logic.
- **Likely tools**: A JS/TS package manager (`pnpm` / `npm` / `yarn` — match the project), TypeScript, the framework's test runner (`vitest` / `jest`), an E2E tool (`playwright` / `cypress`).
- **Encouraged skills/MCPs**: any frontend-focused skill the user has installed; `playwright` MCP for E2E; design-system / shadcn skills if present.
- **Forbidden**: editing deploy / CI workflow files without explicit user OK (see common guards).
- **Commit + self-check**: Conventional Commits. After commit, run the app-scoped test command and report a one-line summary to the lead via `SendMessage`. Branch from the project's default branch.
- **Defaults**: model `sonnet`, permission inherited from lead.

## Role: backend

- **Responsibilities**: server, API, persistence layer.
- **Likely tools**: language-appropriate package manager (`uv` / `pip` / `poetry` for Python, `pnpm` / `npm` for Node, `cargo` for Rust, etc.), framework conventions (FastAPI, Express, NestJS, Django, Spring, etc.), DB migration tooling.
- **Encouraged skills/MCPs**: any backend-specific skill the user has installed; `supabase` MCP, `postgres` MCP, etc., if relevant.
- **Forbidden**: editing deploy / CI workflow files without explicit user OK.
- **Commit + self-check**: Conventional Commits. Run the service-scoped test command after commit and report a one-line summary to the lead. Branch from the project's default branch.
- **Defaults**: model `sonnet`, permission inherited from lead.

## Role: qa

- **Responsibilities**: regression / smoke / E2E tests. Can create new test files. Does NOT edit production code — reports findings instead. Adds cross-scope integration tests on top of each implementer's self-checks.
- **Likely tools**: `vitest` / `jest` (frontend), `pytest` / `unittest` (Python), `playwright` (E2E), `curl` / HTTP clients for API smoke.
- **Encouraged skills/MCPs**: `playwright` MCP; verification skills the user has installed.
- **Output format (required)**: per-scope `✅` / `❌` table. On failure include: reproduction command, ≤5 log lines, one-line suspected cause. Final verdict line `전체 ✅` / `❌ N건` or English equivalent — this verdict gates the optional `automation: pr-create` flow.
- **Commit policy**: test files only (`test(scope): description`). Must not commit production-code changes.
- **Defaults**: model `sonnet`, permission `acceptEdits` recommended (Bash auto-approve OK; blocks new production file edits).

## Role: reviewer

- **Responsibilities**: cross-review other teammates' changes. Verify safety, consistency, project conventions.
- **Likely tools**: `Read`, `Bash(git diff:*, git log:*)`.
- **Encouraged skills/MCPs**: `greptile` MCP if present; security-review tooling; the user's installed reviewer skills.
- **Checklist (universal subset)**: ① commit message convention, ② edits to deploy/CI files without owner approval, ③ secrets in commits, ④ boolean-default safety, ⑤ wording / behavior consistency across changed surfaces.
- **Output format (required)**: per-change `✅` / `⚠️` / `❌` with one-line justification. Final `SendMessage` to lead opens with three lines:
  ```
  reviewer verdict: ❌ N건 / ⚠️ M건 / ✅ K건
  ❌: <one line each>
  ⚠️: <one line each>
  ```
  The lead uses this to gate `automation: pr-create` and inject a reviewer-summary banner in the PR body when configured.
- **Defaults**: model `sonnet`, read-only permission.
- **⚠️ Spawn guard**: do NOT spawn the reviewer via a `subagent_type` that limits `SendMessage` / `TaskUpdate` — inline-inject this role body into the teammate spawn prompt instead. This guard exists because team-collaboration tools are required and external-subagent tool restrictions have caused regressions.

## Role: researcher

- **Responsibilities**: read-only domain mapping. Read the codebase, document "where is X handled, how is Y called", surface 1-2 open questions.
- **Likely tools**: `Read`, `Glob`, `Grep`, `Bash(ls:*, cat:*, git log:*)` (non-destructive only).
- **Encouraged skills/MCPs**: any `probe-*` / explorer skill the user has installed; `context7` MCP for external library docs.
- **Forbidden**: file modification, creation, commit, push.
- **Output format**: one-paragraph summary + file path / line citations + 1-2 open questions.
- **Defaults**: model `sonnet` or `haiku` (lighter work), read-only permission.

## Role: architect

- **Responsibilities**: review the plan itself; identify task dependencies, ordering issues, missing tasks. Advise the lead to refine the plan before spawning the team.
- **Likely tools**: `Read`, `Glob`, `Grep`.
- **Output format**: ① dependency graph (text), ② missing-task candidates, ③ scope-split suggestions.
- **Forbidden**: code modification.
- **Defaults**: model `sonnet`, read-only permission.

## Role: devops (optional)

- **Responsibilities**: CI/CD workflows, deploy manifests, infrastructure-as-code, env-var management.
- **Likely tools**: `gh` CLI, the project's deploy CLI (e.g. `vercel`, `gcloud`, `kubectl`, `terraform`), `task` runners.
- **Forbidden by convention**: editing destructive infra without explicit user approval. The lead must confirm before this role touches production environments.
- **Defaults**: model `sonnet`, permission inherited from lead.

## Role: devils-advocate (optional)

- **Responsibilities**: deliberately challenge other teammates' reports / hypotheses. Surface missing perspectives, exaggerated claims, unvalidated assumptions — one or two per target.
- **Likely tools**: `Read`. Primary activity is `SendMessage`.
- **Forbidden**: code modification; infinite back-and-forth (one round of challenge is sufficient).
- **Output format**: one `SendMessage` per target + a "challenge round complete" signal to the lead.
- **Defaults**: model `sonnet`, read-only permission.
- **⚠️ Spawn guard**: same as reviewer — inline-inject the role body; do not invoke via a restrictive external subagent.

---

## Common guards (auto-injected into every teammate spawn prompt)

These rules are platform-neutral and project-neutral. The lead auto-injects them into every spawn prompt; the user can extend them by adding a `## Project guards` section to their own ROLES.md fork.

1. **Commit convention**: Conventional Commits (`<type>(scope): description`). `description` language defaults to English but the lead may use the user's project convention (e.g. Korean, Japanese) if the plan or repo signals one.
2. **Branching**: branch from the project's default branch (`main` / `master` / `trunk`). Do not branch from a development branch.
3. **CI / deploy files**: do not edit `.github/workflows/**`, `Dockerfile`, IaC manifests, or deploy automation files unless the plan explicitly assigns this work. When in doubt, surface to the user instead of editing.
4. **Package manager respect**: do not mix package managers in a project. Detect the existing manager from lockfiles (`pnpm-lock.yaml`, `package-lock.json`, `uv.lock`, `poetry.lock`, etc.) and use it.
5. **Secrets**: never commit, log, or include in prompts: `.env`, credentials, JWTs, API keys, tokens.
6. **Boolean defaults**: when a field is absent, default to the safer value (e.g. `is_admin` → `False`). Avoid double negatives in identifiers.
7. **No auto-push / no auto-merge**: teammates commit only. Push, PR creation (except gated `automation: pr-create`), and merge are human decisions.
8. **Teammate spawn guard**: do not invoke `reviewer`, `devils-advocate`, or any role that uses team-collaboration tools via a `subagent_type` reference that restricts those tools. Inline-inject the role body from this ROLES.md instead.

---

## Skill discovery

This plugin does not ship a skill index. The lead is expected to discover skills available in the user's environment:

- Project-level skills at `.claude/skills/` (relative to the project root)
- User-level skills at `~/.claude/skills/`
- Plugin-installed skills surfaced via the standard Claude Code skill list

When the lead identifies an obviously relevant skill (e.g. a `probe-*` skill for a debug task), it should recommend or invoke that skill from L3 Enrichment or from a teammate spawn prompt. The lead must not invent skill names.
