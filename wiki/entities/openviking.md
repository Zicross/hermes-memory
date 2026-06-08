---
title: OpenViking
created: 2026-06-08
updated: 2026-06-08
type: entity
tags: [agent-memory, provider, self-hosted]
sources: [raw/articles/agent-memory-landscape-research-2026.md]
confidence: high
contested: false
---

# OpenViking

Open-source context database (AGPLv3) by Volcengine/ByteDance using a **filesystem paradigm** for agent context management.

## Core concept: Context as a filesystem
Everything in a `viking://` URI hierarchy:
```
viking://resources/     ← project docs, repos, web pages
viking://user/          ← preferences, habits
viking://agent/         ← skills, instructions, task memories
```
Agent uses filesystem commands (`ls`, `find`, `grep`) translated to retrieval operations.

## L0/L1/L2 tiered context loading
| Level | Content | Size | Use |
|---|---|---|---|
| L0 | One-sentence abstract | ~100 tokens | Always loaded, relevance check |
| L1 | Core info + usage | ~2K tokens | Planning phase |
| L2 | Full content | Full | Deep dive, on demand |

This explains the 63–91% token reduction in benchmarks.

## Benchmarks (LoCoMo, May 2026)
| Agent | Without | With OpenViking | Accuracy Δ | Latency Δ | Token Δ |
|---|---|---|---|---|---|
| Hermes | 33% | **82.86%** | +2.48× | -66% | -34% |
| OpenClaw | 24% | 82.08% | +3.39× | -59% | -91% |
| Claude Code | 57% | 80.32% | +1.40× | -58% | -63% |

The Hermes result (33% → 82%) is compelling.

## Security/jurisdiction
Volcengine is ByteDance (China). **Can be isolated**: supports OpenAI international endpoints and Ollama. No mandatory outbound calls to Chinese infrastructure when local models used. AGPLv3 license creates downstream concerns for product use.

## Prototype verdict
Strong candidate after Honcho/Mem0. The Hermes benchmark result is noteworthy. AGPLv3 and Volcengine origin require explicit review for sensitive work.

Related: [[external-memory-providers]]
