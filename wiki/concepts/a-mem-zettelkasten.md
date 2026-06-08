---
title: A-MEM (Zettelkasten-Inspired Agent Memory)
created: 2026-06-08
updated: 2026-06-08
type: concept
tags: [agent-memory, academic, wiki]
sources: [raw/articles/agent-memory-landscape-research-2026.md]
confidence: medium
contested: false
---

# A-MEM (Zettelkasten-Inspired Agent Memory)

arXiv:2502.12110 (NeurIPS 2025). "A-MEM: Agentic Memory for LLM Agents" by AGI Research team.

## What it is
Zettelkasten-inspired memory where each note has:
- Unique ID (like a slip)
- Contextual description (auto-generated)
- Keywords and tags
- Links to related notes (auto-discovered)
- Evolution history

**Key innovation**: When a new memory is added, an LLM agent scans existing notes for semantic similarity and creates bidirectional links. Knowledge network emerges bottom-up.

## Hermes relevance
Closest to what an evolved LLM Wiki looks like with automatic cross-linking. The "notes that link themselves" pattern is a natural extension of the existing wiki entity files. Not a product to install — a pattern to implement within the existing wiki tooling.

Related: [[llm-wiki-pattern]], [[layered-agent-memory]]
