# Memory System Maintenance Loop

## Weekly

- Review built-in memory pressure.
- Check `SYSTEM_BOOT.md`, `model-policy.md`, and this repo for stale pointers.
- Review repeated-task detector output for skill candidates.
- Patch umbrella skills when procedures drift.

## Monthly

- Run the recall test suite.
- Save results locally in the repo or Obsidian-style review note.
- Check whether external provider prototype still adds value.
- Remove or archive stale low-value docs.

## After meaningful tasks

Run post-task reflection:

1. Durable compact fact? Built-in memory.
2. Reusable workflow/pitfall? Skill.
3. Architecture/decision/project context? Markdown repo.
4. Historical detail only? Session search.
5. Automation opportunity? Backlog or cron proposal.

## Guardrails

- Quiet jobs by default.
- No recursive cron creation.
- No secrets in outputs.
- No broad self-editing of config without explicit scope, backup, and verification.
- External memory provider health checks should alert only when broken or materially stale.
