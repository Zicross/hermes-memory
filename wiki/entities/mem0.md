---
title: Mem0
created: 2026-06-08
updated: 2026-06-08
type: entity
tags: [agent-memory, provider, self-hosted]
sources: [raw/articles/agent-memory-landscape-research-2026.md]
confidence: high
contested: false
---

# Mem0

Universal memory layer (Apache 2.0, YC S24) with self-hosted docker-compose and the v3 ADD-only algorithm.

## v3 Algorithm (April 2026)
Single-pass ADD-only extraction — no UPDATE/DELETE operations. Memories accumulate; temporal retrieval surfaces the most recent/relevant fact. Entity linking for retrieval boosting. Multi-signal retrieval: semantic + BM25 + entity + temporal.

## Benchmarks (self-reported)
- LongMemEval: **94.8** (claimed, April 2026)
- LoCoMo: **91.6** (+20pp over prior algorithm)
- BEAM 1M: 64.1; BEAM 10M: 48.6

**Caveat**: self-reported on own eval framework. Backbone LLM not specified. Not directly comparable to Hindsight (91.4, Gemini-3) or Honcho numbers.

## Self-host
```bash
cd server && docker compose up  # FastAPI + Postgres+pgvector
# Default LLM: gpt-5-mini (configurable to Anthropic via LiteLLM)
```

## Architecture
- Fact extraction per message batch via LLM
- Deduplication at retrieval time (v3: not at write time)
- Optional Neo4j/FalkorDB/Kuzu graph backend (+2% accuracy)

## Hermes fit
Strong second prototype. Apache 2.0 license (vs Honcho's AGPL-3.0). Simpler ADD-only algorithm. Broader ecosystem (LangChain, LangGraph, CrewAI).

Related: [[external-memory-providers]], [[honcho]], [[hindsight]]
