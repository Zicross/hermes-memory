# Hermes Memory System Research + Architecture Proposal

**Date:** 2026-06-08T00:44:34Z  
**Status:** Phase 0–2 complete; prototype not started  
**Owner:** Isaac / Zicross  
**Source handoff:** `handoffs/2026-06-08-memory-system-buildout-handoff.md` and local `~/.hermes/planning/handoffs/2026-06-08-memory-system-buildout-handoff.md`

## Executive Summary

Hermes should **not** jump straight to Honcho, Mem0, or a raw vector DB. The right architecture is layered:

1. **Live config / tools** for current state.
2. **Boot context** for small always-visible operational pointers.
3. **Built-in Hermes memory** for compact durable facts/preferences only.
4. **Session search** for historical transcript recall.
5. **Skills** for reusable procedures and pitfalls.
6. **Human-readable markdown knowledge repo / Obsidian-style notes** for source-of-truth architecture, decisions, research, and project context.
7. **LLM Wiki / compiled knowledge layer** for learning-heavy research: raw sources → interlinked entity/concept/comparison pages with provenance, contradictions, and update logs.
8. **External semantic memory** for associative recall/user modeling after the base loop is reliable.
9. **Cron / maintenance loops** for hygiene, consolidation, and recall tests.

**Provider decision:** Defer a permanent production provider. Run a limited A/B prototype with **Honcho self-host first** and **RetainDB Local second**. Honcho is the best fit for user/agent modeling and Hermes has first-class provider support. RetainDB Local is promising for coding-agent/project-memory workflows with low setup friction and no cloud/API requirement. Hindsight is strong but heavier and more product/API-like. Mem0 and Supermemory are credible hosted/open-source options but should not become source of truth before the markdown/skills/session layers are disciplined.

## Phase 0 — Current Hermes Inventory

Verified live on 2026-06-08:

- `hermes memory status`: built-in memory active; no external provider configured.
- Installed memory plugins: ByteRover, Hindsight, Holographic, Honcho, Mem0, OpenViking, RetainDB, Supermemory.
- `~/.hermes/config.yaml`:
  - Primary: `openai-codex / gpt-5.5`
  - Fallback: direct `anthropic / claude-sonnet-4-6` only
  - `memory.provider: ""`
  - Built-in memory/user profile enabled
  - Built-in memory char limit: 2200
  - User profile char limit: 1375
  - Aux models use verified-free OpenRouter routes such as `nvidia/nemotron-3-super-120b-a12b:free`; vision currently `google/gemma-4-31b-it:free`
  - `agent.tool_use_enforcement: strict`
  - `provider_routing.require_parameters: true`, `data_collection: deny`
- `hermes status --all`:
  - Gateway running under user systemd
  - Discord configured
  - 9 active cron jobs
- Existing brain/maintenance crons:
  - Weekly Hermes Brain Review, local delivery
  - Repeated-Task Detector, local delivery
  - Usage/Codex/model review jobs

Canonical source-of-truth files for model/memory decisions:

- `~/.hermes/config.yaml`
- `~/.hermes/SYSTEM_BOOT.md`
- `~/.hermes/model-policy.md`
- `~/.hermes/planning/handoffs/2026-06-08-memory-system-buildout-handoff.md`
- `/home/hermes/claude-code-memory/`
- `model-management`, `hermes-agent`, `hermes-memory-providers`, and agent-brain skills

## Phase 0.5 — OpenClaw Audit Findings

Mounted paths inspected:

- `/mnt/openclaw-engine` — OpenClaw engine/package tree, dist JS, memory plugins, memory-core, memory-lancedb, etc.
- `/mnt/openclaw-runtime` — smaller runtime/template workspace with stock-looking files.
- `/mnt/openclaw` — real useful “Big Z” workspace with memory, deliverables, projects, scripts, skills, reports.
- `/root/openclaw-audit/state` and `/root/openclaw-audit` were not present in this container despite being mentioned in the handoff.

Important distinction:

- `/mnt/openclaw-runtime/data/` looks mostly like stock/template or secondary runtime state.
- `/mnt/openclaw/` is the high-value workspace to mine.

OpenClaw memory documents found and inspected:

- `/mnt/openclaw/memory/resources/openclaw-memory-technical-reference.md`
- `/mnt/openclaw/memory/archive/research-2026-03/memory-architecture-final-report-2026-03-24.md`
- `/mnt/openclaw/memory/projects/memory-architecture-proposal.md`

### What OpenClaw did well

OpenClaw’s effective pattern was **markdown as source of truth + searchable index**:

- Always-loaded bootstrap files: `AGENTS.md`, `SOUL.md`, `USER.md`, `IDENTITY.md`, `TOOLS.md`, `HEARTBEAT.md`, `BOOT.md`, `MEMORY.md`.
- Searchable knowledge under `memory/` with PARA-ish structure:
  - daily notes
  - `projects/`
  - `areas/`
  - `resources/`
  - `archive/`
- Memory search indexed `MEMORY.md` and `memory/**/*.md`.
- Search backend used hybrid BM25 + vector, with QMD described as local-first sidecar, optional MMR, and temporal decay.
- OpenClaw’s own research converged on a 3-layer architecture:
  - small bootstrap index / cheat sheet
  - searchable markdown KB
  - daily notes + consolidation/promotion loop

### OpenClaw lessons to keep

1. **Markdown files should remain human-readable source of truth.** Semantic memory is a retrieval aid, not the authority.
2. **Always-loaded memory must be an index, not a warehouse.** Hermes’ 2,200-char memory is even tighter than OpenClaw’s `MEMORY.md`; it should hold pointers only.
3. **Daily/session logs need promotion.** Raw logs decay into junk without a consolidation loop.
4. **Hybrid retrieval matters.** Pure vector search loses exact config/model/path facts; combine lexical + semantic + metadata.
5. **Temporal decay helps, but evergreen files must be exempt.** Policies and source-of-truth docs should not decay like daily notes.
6. **Separate current state from historical memory.** Live tools beat stale notes.

### OpenClaw pitfalls to avoid

1. **Plaintext secrets in config/transcripts.** Hermes should keep secrets in `~/.hermes/.env` and avoid copying them into notes or chat.
2. **Config self-edit instability.** OpenClaw had repeated config clobbering/truncation; Hermes memory automation should use atomic writes/backups and avoid broad self-editing.
3. **Split brain locations.** OpenClaw had confusing overlap between template runtime data and real workspace. Hermes needs one obvious brain root plus pointers.
4. **Surface sprawl.** Too many Discord channels/personas increase token burn and reduce coherence.
5. **Memory junk drawers.** `lessons-learned.md` style files need promotion/classification or they become unsearchable noise.

## Phase 1 — Provider / Store Comparison

### Honcho

Primary-source findings: Honcho describes itself as memory infrastructure for stateful agents that understand people, agents, groups, projects, and ideas over time. It stores messages/events, reasons in the background, and exposes peer representations, session context, search, and natural-language insight. It supports managed `api.honcho.dev` and self-hosted FastAPI.

Fit for Hermes:

- Best conceptual fit for **user modeling / agent modeling**.
- Hermes already has Honcho provider/plugin support and `hermes memory setup`/`hermes honcho setup` paths.
- Good first external semantic-memory prototype.

Risks:

- Extra service to operate.
- Need credential/base URL management.
- Should not become the only source of truth.

### RetainDB

Primary-source findings: RetainDB offers local and server/cloud modes. Local starts with `npx -y @retaindb/local`, provides API/viewer, MCP tools, BM25 + vector + graph retrieval, memory types, consolidation, reinforcement, context packs, and local storage under `~/.retaindb/`.

Fit for Hermes:

- Very strong candidate for **coding-agent/project memory**.
- Low-friction local mode, no Postgres required for local.
- Good for cross-tool coding context (Codex/Claude/OpenCode-style workflows).

Risks:

- Newer and may overlap with Hermes’ own session/skill/memory responsibilities.
- Need to test Hermes provider maturity specifically.

### Hindsight

Primary-source findings: Hindsight is an agent memory system focused on agents learning over time, not just recalling conversation history. Docker quickstart exposes API and UI; supports multiple LLM providers.

Fit:

- Strong learning/reinforcement framing.
- Self-hosted Docker path exists.

Risks:

- Heavier stack.
- More product/service-like than a simple Hermes brain layer.
- Needs LLM API key/provider wiring.

### Mem0

Primary-source findings from search/docs: Mem0 is a universal memory layer with hosted and open-source/self-hosted options. It emphasizes persistent context across sessions, fact extraction, semantic search, dedupe/reranking.

Fit:

- Mature, well-known memory abstraction.
- Good baseline external provider.

Risks:

- Can become “black box fact extractor” if not paired with explicit write policy.
- Less obviously agent/persona-centric than Honcho for Isaac/Hermes.

### Supermemory

Primary-source findings: Supermemory positions itself as memory/context layer with automatic learning from conversations, fact extraction, user profiles, hybrid search, connectors, multimodal extractors, and plugins for Hermes/OpenClaw/Claude Code.

Fit:

- Strong hosted product and connector story.
- Good for broad personal/company brain and external app integrations.

Risks:

- More cloud/product oriented.
- Adds another hosted dependency unless self-hosting is mature enough for our use case.

### OpenViking

Primary-source findings: OpenViking is an open-source context database for AI agents, using a filesystem paradigm with L0/L1/L2 tiered context loading, recursive retrieval, trajectory observability, and self-iteration.

Fit:

- Conceptually close to OpenClaw/Hermes layered context.
- Strong “context filesystem” framing.

Risks:

- Origin/jurisdiction: Volcengine/ByteDance ecosystem. Avoid for China-sensitive work and do not make it core Hermes memory without explicit review.
- More experimental/complex.

### ByteRover

Primary-source findings: ByteRover CLI provides persistent structured memory for coding agents via context tree, web UI, git-like versioning, review workflow, cloud sync, MCP, and multiple coding-agent integrations.

Fit:

- Strong coding-project context tree and review workflow.
- Good for curated project memory across tools.

Risks:

- Likely overlaps with the shared markdown repo and skills.
- Cloud sync/API key path may add friction.
- Best evaluated later after base markdown loop is stable.

### Raw vector / graph stores

- **Qdrant / Chroma / LanceDB / pgvector**: useful infrastructure if building our own retrieval service. They do not solve memory classification, promotion, stale fact management, or authority policy by themselves.
- **Neo4j / Kuzu / graph-style stores**: useful once relationships matter and there are enough entities/claims to justify graph maintenance. Too much friction for the first prototype.

## Phase 2 — Proposed Architecture

### Source-of-truth hierarchy

1. **Live state**: `hermes status`, `hermes config`, `hermes memory status`, filesystem, git, cron list.
2. **Config files**: `~/.hermes/config.yaml`, `.env` for secrets only, actual repo/project configs.
3. **Boot/policy docs**: `~/.hermes/SYSTEM_BOOT.md`, `~/.hermes/model-policy.md`.
4. **Shared markdown memory repo**: `/home/hermes/claude-code-memory/` and GitHub `Zicross/hermes-memory`.
5. **Skills**: reusable procedures and pitfalls.
6. **Session search**: historical evidence and exact past conversation reconstruction.
7. **Built-in memory**: small pointers/preferences only.
8. **External semantic memory**: associative hints/context, never sole authority.

### Write-routing policy

- **Built-in memory:** stable, compact behavior-shaping facts and pointers only.
- **Skills:** reusable procedures, commands, pitfalls, verification recipes.
- **Shared markdown repo / Obsidian-style notes:** curated architecture, decisions, project context, provider research, handoffs.
- **Session search:** historical record; don’t duplicate one-off task history into memory.
- **External semantic provider:** conversation/user/agent embeddings and associative recall after source-of-truth layer is clean.
- **Cron:** review and hygiene; not authority.

### Proposed repo structure additions

```text
/home/hermes/claude-code-memory/
├── context/current.md
├── decisions/log.md
├── handoffs/
├── projects/
├── workflows/
├── research/
│   └── 2026-06-08-hermes-memory-system-research-and-architecture.md
├── memory-system/
│   ├── architecture.md
│   ├── write-routing-policy.md
│   ├── recall-test-suite.md
│   ├── provider-evaluation.md
│   ├── llm-wiki-integration.md
│   └── maintenance-loop.md
└── wiki/                       # optional LLM Wiki layer for compiled research
    ├── SCHEMA.md
    ├── index.md
    ├── log.md
    ├── raw/
    ├── entities/
    ├── concepts/
    ├── comparisons/
    └── queries/
```

## Phase 3 — Prototype Plan

Do **not** install/configure a provider as part of this research pass. Prototype in this order:

1. **Markdown authority layer first**
   - Create the `memory-system/` docs above.
   - Update `context/current.md` and `decisions/log.md` with current model/memory source-of-truth pointers.
   - Add explicit retrieval order.

2. **LLM Wiki learning layer**
   - Create a small wiki under `/home/hermes/claude-code-memory/wiki/` with `SCHEMA.md`, `index.md`, and `log.md`.
   - Ingest raw provider/OpenClaw sources with provenance and hashes.
   - Compile entity/concept/comparison pages so Isaac can learn from the same durable artifact Hermes uses.
   - Keep the wiki authoritative for compiled research only; live config still comes from tools/config.

3. **Recall test suite**
   - Fresh-session questions:
     - What is current model/fallback/OpenRouter policy?
     - Where is source of truth?
     - What goes into built-in memory vs skills vs docs vs session search?
     - What was learned from OpenClaw?
     - Which memory provider was selected/deferred and why?
     - How is recall tested without hallucinating stale state?
   - Each answer must cite the authority source it used.

4. **Honcho self-host prototype**
   - Use official Hermes setup path: `hermes memory setup honcho` or `hermes honcho setup` after self-host service exists.
   - Store secrets/base URLs in `.env` / config, not notes.
   - Ingest only a small non-sensitive test corpus first.
   - Test whether it improves recall beyond markdown + session search.

5. **RetainDB Local comparison**
   - Start local RetainDB separately.
   - Test project/coding context recall and handoff generation.
   - Compare friction, precision, observability, and source-of-truth compatibility against Honcho.

6. **Decision gate**
   - Adopt one external provider only if it passes recall tests and does not create authority confusion.
   - Otherwise keep built-in + markdown + session search + skills, and defer external semantic memory.

## Phase 4 — Verification / Rollout

Acceptance checks:

- A fresh Hermes session can locate the handoff and this report.
- A fresh session answers the six recall-test questions with citations to live/config/docs.
- Built-in memory remains compact and does not exceed limits with full architecture prose.
- No secrets appear in markdown repo, skills, or memory summaries.
- Provider configuration, if later enabled, passes `hermes memory status` after gateway restart/new session.
- External provider output is treated as hints unless corroborated by config/docs/live tools.

## Phase 5 — Maintenance Loop

Recommended quiet jobs:

- Weekly memory-system review: check stale pointers, memory pressure, docs consistency, skill gaps.
- Monthly recall test: run the six fresh-session questions and save results locally.
- Post-task reflection loop: classify durable facts vs skills vs docs vs session-only history.
- Provider health check if Honcho/RetainDB/etc. is enabled.

## Immediate Next Actions

1. Create `memory-system/architecture.md`, `write-routing-policy.md`, `recall-test-suite.md`, `provider-evaluation.md`, and `maintenance-loop.md` from this report.
2. Update `context/current.md` and `decisions/log.md` so future Claude/Gemma/Hermes sessions see the memory-system state.
3. Commit and push to `github.com/Zicross/hermes-memory`.
4. Only after that, decide whether to start Honcho self-host prototype.
