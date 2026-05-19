---
description: Execute an approved recipe in team / single / manual mode (step 3 of assemble-team)
argument-hint: <execution recipe markdown | path to recipe file>
allowed-tools: Skill, Read, Glob, Grep, Bash, AskUserQuestion, Agent, TeamCreate, TeamDelete, TaskList, SendMessage
---

Invoke the `execute-work` skill from the assemble-team plugin on the execution recipe provided in the arguments. The input must be a recipe (frontmatter contains `recipe_id` and `mode`):

$ARGUMENTS
