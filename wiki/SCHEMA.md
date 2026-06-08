# Wiki Schema

## Domain

Hermes long-term memory architecture, agent memory systems, OpenClaw lessons, external semantic memory providers, and compiled research for building Isaac's best possible agent brain.

## Conventions

- File names: lowercase, hyphenated, no spaces.
- Every wiki page starts with YAML frontmatter.
- Use wikilinks for cross-references.
- Every new page must appear in `index.md`.
- Every action must be appended to `log.md`.
- Raw sources in `raw/` are immutable and include `source_url`, `ingested`, and `sha256` frontmatter.
- Provider claims should cite raw source files or the research report.
- Live Hermes config claims must be verified with tools/config, not inferred from wiki pages.

## Frontmatter

```yaml
---
title: Page Title
created: YYYY-MM-DD
updated: YYYY-MM-DD
type: entity | concept | comparison | query | summary
tags: [agent-memory]
sources: [raw/articles/example.md]
confidence: high | medium | low
contested: false
---
```

## Tag Taxonomy

- agent-memory
- hermes
- openclaw
- provider
- semantic-memory
- local-first
- hosted
- self-hosted
- vector-search
- hybrid-retrieval
- graph-memory
- wiki
- source-of-truth
- privacy
- coding-agents
- benchmark
- comparison

## Authority Policy

The wiki is authoritative for compiled research synthesis only. Runtime facts come from live tools and config. Final decisions are promoted to `../memory-system/` and `../decisions/log.md`.
