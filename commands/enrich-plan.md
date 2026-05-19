---
description: Enrich a plan with classification + ambiguity scoring + grilling (step 1 of assemble-team)
argument-hint: <plan body | path to plan file | GitHub issue/PR URL>
allowed-tools: Skill, Read, Glob, Grep, Bash, AskUserQuestion, WebFetch
---

Invoke the `enrich-plan` skill from the assemble-team plugin on the plan body, path, or URL provided in the arguments. Pass the user input through verbatim:

$ARGUMENTS
