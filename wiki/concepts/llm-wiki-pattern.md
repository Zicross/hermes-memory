---
title: LLM Wiki Pattern (Karpathy)
created: 2026-06-08
updated: 2026-06-08
type: concept
tags: [wiki, agent-memory, source-of-truth]
sources: [raw/articles/agent-memory-landscape-research-2026.md]
confidence: high
contested: false
---

# LLM Wiki Pattern (Karpathy)

A **pattern** (not a product) for maintaining a compounding agent knowledge base as interlinked Markdown files. Published by Andrej Karpathy (~April 2026, gist `442a6bf555914893e9891c11537de94f`).

## Three operations
1. **Ingest** — LLM reads source, creates or amends a wiki article. Content is distilled into editable Markdown with source links and cross-references.
2. **Query** — At runtime, wiki index (titles + one-line summaries) loaded into context. Agent reads specific articles on demand. Named queries = reusable prompt templates for common lookups.
3. **Lint** — Contradiction detection between articles. Flags or merges conflicting facts.

## Why it beats RAG for durable knowledge
| Dimension | RAG | LLM Wiki |
|---|---|---|
| Representation | Raw chunks | Distilled human-editable prose |
| Updates | Re-embed source | Edit one file; re-lint |
| Contradictions | Silently coexist | Flagged by lint pass |
| Inspectability | Black box | Git-diff-able |
| Compounding | None | Each ingest enriches the whole KB |

## Hermes relevance
**The Hermes wiki at `/home/hermes/claude-code-memory/wiki/` IS already this pattern.** The next step is formalizing ingest/lint loops and growing the library.

This layer is **complementary to external semantic memory** — not competing. The wiki is the compiled human-readable knowledge layer; Honcho/Mem0 is the associative recall/user-modeling layer.

Related: [[layered-agent-memory]], [[external-memory-providers]]
