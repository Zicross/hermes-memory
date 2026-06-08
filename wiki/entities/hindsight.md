---
title: Hindsight
created: 2026-06-08
updated: 2026-06-08
type: entity
tags: [agent-memory, provider, self-hosted]
sources: [raw/articles/agent-memory-landscape-research-2026.md]
confidence: high
contested: false
---

# Hindsight

Biomimetic agent memory (MIT license) focused on making agents **learn over time**, not just remember conversations.

## Architecture: World / Experiences / Mental Models
Three memory types modeled on human cognition:
- **World**: objective facts about the world
- **Experiences**: the agent's own history
- **Mental Models**: synthesized understanding from reflecting on World + Experiences

## Retrieval: 4 strategies in parallel
Semantic (dense vectors) + BM25 (keyword) + Graph (entity/temporal/causal links) + Temporal (time range). Fused via RRF + cross-encoder rerank.

**Reflect operation**: deeper reasoning pass over memories that generates new insights and connections. Closest to Honcho's dialectic reasoning.

## Benchmarks
- **LongMemEval: 91.4%** (with Gemini-3 Pro backbone)
- **Independently validated** by Virginia Tech Sanghani Center and The Washington Post
- With OSS-120B: 89%

## Self-host
```bash
docker run -it --pull always --name hindsight --restart unless-stopped \
  -p 8888:8888 -p 9999:9999 \
  -e HINDSIGHT_API_LLM_API_KEY=<key> \
  -v hindsight-data:/home/hindsight/.pg0 \
  ghcr.io/vectorize-io/hindsight:latest
# API: :8888, UI: :9999
# Supports: openai, anthropic, gemini, ollama, lmstudio
```

## Hermes fit
Third candidate. Single docker run, Anthropic-compatible, Ollama for fully local. The `reflect` operation is compelling. Heavier than needed at current scale.

Related: [[external-memory-providers]], [[honcho]], [[mem0]]
