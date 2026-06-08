---
title: Supermemory
created: 2026-06-08
updated: 2026-06-08
type: entity
tags: [agent-memory, provider, hosted, benchmark]
sources: [raw/articles/agent-memory-landscape-research-2026.md]
confidence: high
contested: false
---

# Supermemory

Memory and context platform with dedicated Hermes plugin. Claims #1 on LongMemEval/LoCoMo/ConvoMem.

## Benchmark reality check
- **LongMemEval: 81.6%** (self-reported on their own MemoryBench framework)
- Claims #1 on LoCoMo and ConvoMem — no % published
- Mem0 v3 claims 94.8% on LongMemEval (same benchmark, different setup)
- Benchmark claims are **contested** — other providers (Mem0, Hindsight) do not accept their MemoryBench rankings as authoritative

## Architecture
Memory engine (fact extraction, updates, auto-forgetting of expired info) + User Profiles (static + dynamic) + Hybrid Search (RAG + memory) + Connectors (Google Drive, Gmail, Notion, OneDrive, GitHub) + File Processing (PDFs, images, videos, code).

**Automatic forgetting** of temporary facts (e.g. "exam tomorrow") after the date passes.

## Self-host
Exists but is **enterprise-only / not documented** as a simple docker-compose. Cloud dependency for standard users.

## Hermes fit
Rapid cloud testing via dedicated Hermes plugin (`hermes memory setup`). Lower LongMemEval score than Mem0 and Hindsight. Self-host situation unclear.

Related: [[external-memory-providers]]
