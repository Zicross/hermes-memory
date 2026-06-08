---
title: ByteRover
created: 2026-06-08
updated: 2026-06-08
type: entity
tags: [agent-memory, provider, coding-agents, local-first]
sources: [raw/articles/byterover-readme.md]
confidence: high
contested: false
---

# ByteRover

AI coding agent memory CLI with persistent structured context tree, git-like versioning, and cloud sync.

## Key benchmarks (from their arXiv paper 2604.01599)
- LoCoMo: **96.1%** (1,982 questions)
- LongMemEval-S: **92.8%** (500 questions, 23,867 docs)

**These were misattributed to RetainDB in the original provider evaluation — they belong to ByteRover.**

## Architecture
Context tree with tiered file-search retrieval (fuzzy text → LLM-driven deeper search). 20 LLM providers. 24 built-in agent tools. Cloud sync with push/pull. Git-like branch/commit/merge for context versions.

## Usage
```bash
npm install -g byterover-cli
cd your/project && brv
/curate "Auth uses JWT with 24h expiry" @src/middleware/auth.ts
/query How is authentication implemented?
```

## Hermes fit
Promising for coding-project context trees. Evaluate after shared markdown repo loop stabilizes.

Related: [[external-memory-providers]], [[retaindb]]
