---
title: Honcho
created: 2026-06-08
updated: 2026-06-08
type: entity
tags: [agent-memory, provider, self-hosted, user-modeling]
sources: [raw/articles/honcho-research-report.md]
confidence: high
contested: false
---

# Honcho

AI-native memory focused on **peer/session relationship modeling** — modeling who users and agents are over time, not just storing facts.

## Core differentiation
- **Peers** (users, agents, groups, projects, ideas) participate in **sessions**. Background **deriver** processes messages asynchronously to build **representations** and **peer cards**.
- **Dialectic** (`peer.chat("natural question?")`) = LLM-synthesized answer from everything Honcho knows. Qualitatively different from fact retrieval.
- **Multi-peer symmetry**: can model user, agent, and project knowledge separately.
- **Conclusions**: agent can explicitly write persistent facts (`honcho_conclude`). Honcho self-heals incorrect conclusions over time.

## Self-host setup
```bash
git clone https://github.com/plastic-labs/honcho.git && cd honcho
cp docker-compose.yml.example docker-compose.yml && cp .env.template .env
# .env: LLM_ANTHROPIC_API_KEY=..., AUTH_USE_AUTH=***  compose up
# API on :8000
```
Requires: Docker, Postgres+pgvector (included in docker-compose), one LLM API key.
License: **AGPL-3.0**.

## Hermes plugin
5 tools: `honcho_profile`, `honcho_search`, `honcho_reasoning`, `honcho_context`, `honcho_conclude`. Background prefetch fires dialectic queries between turns. Cron guard prevents polluting user model from cron sessions. Config: `~/.hermes/honcho.json`.

## Benchmarks
Does not publish specific LongMemEval/LoCoMo % in README. Claims "Pareto Frontier of Agent Memory" (accuracy vs cost). Scores referenced at honcho.dev/evals — not yet fetched.

## Privacy
Self-hosted: 100% local. Cloud (api.honcho.dev): all data to Plastic Labs. Use self-host.

## Prototype verdict
**First self-host prototype.** Hermes plugin is mature. Self-host path is clear (single docker compose). Dialectic adds capability not present in existing stack. Use same Claude Max OAuth for the deriver.

Related: [[external-memory-providers]], [[retaindb]], [[mem0]]
