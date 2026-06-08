# Memory Write-Routing Policy

## Save to built-in Hermes memory

Use only for compact, durable facts that shape future behavior:

- stable user preferences
- stable environment pointers
- source-of-truth pointers
- recurring corrections

Do not save task completion logs, PR/issue numbers, temporary status, raw research, or long architecture prose.

## Save to skills

Use for reusable procedures and pitfalls:

- setup workflows
- debugging recipes
- config-editing gotchas
- verification checklists
- provider-specific commands

Patch existing umbrella skills before creating narrow one-off skills.

## Save to this markdown repo / Obsidian-style knowledge base

Use for curated human-readable knowledge:

- architecture
- decisions
- project context
- provider research
- handoffs
- source-of-truth maps
- recall test results

## Leave in session search

Use session search for historical reconstruction:

- what happened in a specific conversation
- temporary work state
- exact prior wording
- stale-by-default implementation details

## External semantic provider

Use as associative recall/user modeling only. It should suggest context, not override config/docs/live tools.

## Negative rules

- No secrets in notes, skills, memory, or reports.
- No raw logs unless specifically needed and sanitized.
- No current-state claims without live verification.
- No broad autonomous self-edits of config without backup/verification.
