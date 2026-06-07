# Claude Code Memory Base

> **Repo:** https://github.com/Zicross/hermes-memory

Shared context for Claude Code (Opus → Gemma) across laptop and host machines. Hermes maintains this.

## Structure

- [[context/]] — Current context (active projects, recent decisions)
- [[projects/]] — Project-specific context (one file per project)
- [[workflows/]] — Workflow reference (how we do things)
- [[decisions/]] — Settled decisions log

## How to Use

1. Read [[context/current]] for active context
2. Check [[projects/<project-name>]] for project context
3. Reference [[workflows/coding]] for workflow standards
4. Check [[decisions/log]] before asking "how do we do X"

## Updating

Hermes maintains this. To request an update, tell Hermes what changed.