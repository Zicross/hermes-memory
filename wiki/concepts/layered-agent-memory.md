---
title: Layered Agent Memory
created: 2026-06-08
updated: 2026-06-08
type: concept
tags: [agent-memory, hermes, source-of-truth, wiki]
sources: [raw/articles/openclaw-memory-technical-reference.md, raw/articles/openclaw-memory-final-report-2026-03-24.md]
confidence: high
contested: false
---

# Layered Agent Memory

A durable agent brain should not collapse all context into one store. Hermes should separate live state, config, boot context, built-in memory, session search, skills, markdown knowledge, LLM Wiki compiled research, and optional external semantic memory.

OpenClaw's strongest lesson was that markdown files should remain human-readable source of truth while search/index layers make them retrievable. [[openclaw]] used bootstrap files plus searchable `memory/` markdown. Hermes has tighter built-in memory limits, so its always-visible memory should be only a pointer/index.

Related: [[external-memory-providers]], [[honcho]], [[retaindb]]
