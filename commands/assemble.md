---
description: Run the full enrich → map → execute chain (orchestrator entry point)
argument-hint: <plan body | path to plan file | GitHub issue/PR URL>
allowed-tools: Skill, Read, Glob, Grep, Bash, AskUserQuestion, WebFetch, Agent, TeamCreate, TeamDelete, TaskList, SendMessage
---

Invoke the `assemble` orchestrator skill on the plan body, path, or URL provided in the arguments. The orchestrator chains enrich-plan → verify-mapping → execute-work with validation at each handoff:

$ARGUMENTS
