---
title: RetainDB
created: 2026-06-08
updated: 2026-06-08
type: entity
tags: [agent-memory, provider, local-first, coding-agents, hybrid-retrieval]
sources: [raw/articles/retaindb-research-report.md]
confidence: high
contested: false
---

# RetainDB

Coding-agent memory infrastructure with 12 typed memories, BM25+vector+graph hybrid retrieval, and delta-compressed context packs.

## Architecture
- **Local** (Apache 2.0): `npx -y @retaindb/local` → API on :3111, viewer on :3113. Fully air-gapped, no cloud dependency. Storage: `~/.retaindb/local-store.json`.
- **Server** (BSL 1.1): docker-compose, requires Postgres+pgvector.
- **Cloud**: Managed hosted service, paid.

## Retrieval pipeline
BM25 → vector → graph signals → **RRF fusion** → cross-encoder reranker. Best-in-class hybrid retrieval for exact terminology (function names, error messages).

## Memory types (12)
`factual`, `preference`, `semantic`, `procedural`, `decision`, `constraint`, `instruction`, `goal`, `event`, `correction`, `session_summary`, `project_state`. Consolidation: duplicate merge, session rollups, stale decay.

## Context packs
Token-budgeted payloads with delta compression — agents receive only what changed since last pack. Includes file chunks, code maps, compressed tool output.

## Hermes plugin
**Targets cloud API** — `RETAINDB_API_KEY` is required. Local-mode bridge possible via `RETAINDB_BASE_URL=http://localhost:3111`. 10 tools including file store.

## Benchmarks
**79% LongMemEval overall** (RetainDB's own published numbers). The 96.1%/92.8% numbers belong to ByteRover, not RetainDB.

## Privacy
Local mode: 100% air-gapped. Cloud plugin: all turns sent to api.retaindb.com. Bridge to local runtime for privacy.

## Prototype verdict
Strong second prototype for coding-agent project memory. Node.js v24 already installed. Bridge via `RETAINDB_BASE_URL=http://localhost:3111`.

Related: [[external-memory-providers]], [[honcho]], [[byterover]]
