---
description: Derive team cards + select execution mode (step 2 of assemble-team)
argument-hint: <enriched plan markdown | path to enriched plan file>
allowed-tools: Skill, Read, Glob, Grep, AskUserQuestion, Bash
---

Invoke the `verify-mapping` skill from the assemble-team plugin on the enriched plan provided in the arguments. The input must be an enriched plan (frontmatter contains `forced_blocks_resolved: true`):

$ARGUMENTS
