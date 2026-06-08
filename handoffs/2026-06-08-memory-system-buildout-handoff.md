---
type: handoff
date: 2026-06-08
owner: Isaac / Zicross
subject: Build Hermes long-term memory system properly
status: ready-for-research
priority: high
---

# Handoff: Build Hermes Long-Term Memory System Properly

## Why this exists

Hermes has repeatedly forgotten or misremembered important setup decisions: model routing, what was already configured, workflow ownership, and where canonical files live. The immediate patch is now in place, but it is not a real memory architecture.

Current state is better than before but still fragile:

- Built-in Hermes memory is active but small and should only contain compact durable facts.
- `session_search` can recall past transcripts, but the agent must remember to use it.
- `~/.hermes/SYSTEM_BOOT.md` and `~/.hermes/model-policy.md` now hold canonical operational context.
- `model-management` skill carries reusable model/provider procedures.
- `github.com/Zicross/hermes-memory` is cloned at `/home/hermes/claude-code-memory/` as a shared Claude Code / Opus / Gemma context repo.
- No external semantic memory provider is currently configured (`hermes memory status` reports built-in only).

This handoff is for building the memory system *properly*, research-first, not improvising another ad hoc file pile.

## Non-negotiable operating principle

Do research before building. The first phase is discovery and comparison, not implementation.

The agent taking this handoff must not jump straight to installing Honcho, Mem0, a vector DB, or writing sync scripts. First understand:

1. What Hermes already supports.
2. What OpenClaw did in the previous installation Isaac created.
3. What external/self-hosted memory systems are worth considering.
4. What data should live in which memory layer.
5. How recall will be tested in fresh sessions.

## Current verified Hermes state

As of 2026-06-08:

- Primary model: `openai-codex / gpt-5.5`.
- Fallback: direct `anthropic / claude-sonnet-4-6` only.
- OpenRouter is not in `fallback_providers`.
- OpenRouter key remains active only for verified free auxiliary models.
- Paid OpenRouter guard exists in installed Hermes code; paid OpenRouter model IDs are blocked unless the ID ends in `:free` or equals `openrouter/owl-alpha` / `openrouter/free`.
- Canonical files for model routing:
  - `~/.hermes/config.yaml`
  - `~/.hermes/SYSTEM_BOOT.md`
  - `~/.hermes/model-policy.md`
  - `model-management` skill

Always verify this live before making decisions:

```bash
hermes memory status
hermes config check
hermes status
python ~/.hermes/scripts/openrouter_model_catalog_audit.py <model-id>
```

## Existing memory-related artifacts

### Core Hermes files

- `~/.hermes/SYSTEM_BOOT.md` — boot/operator context; contains current model routing and operating principles.
- `~/.hermes/model-policy.md` — current model routing, OpenRouter free-only policy, memory write-routing policy.
- Built-in memory store — compact durable facts only; currently near capacity and must be curated carefully.
- Session DB / `session_search` — historical transcripts and provenance.

### Shared Claude/Gemma/Opus context repo

- Repo: `github.com/Zicross/hermes-memory`
- Local clone: `/home/hermes/claude-code-memory/`
- Current files:
  - `README.md`
  - `context/current.md`
  - `decisions/log.md`
  - `workflows/coding.md`
  - `projects/constiuint.md`

This repo should become a stable cross-agent context base if that remains the chosen pattern.

### Prior handoff

- CivicBridge repo handoff: `/home/hermes/projects/civicbridge/planning/handoffs/2026-06-07-memory-system-handoff.md`
- That handoff diagnosed the original root cause: relying on a 2,200-character memory block as if it were a full memory system.

## OpenClaw cross-reference requirement

Isaac says he previously created an OpenClaw installation that should be cross-referenced.

Current local quick search did **not** find obvious `openclaw` paths under `/home/hermes`. Do not assume it is absent. It may be:

- on the host machine outside the LXD container,
- in another user directory,
- in a different repo name,
- in an older session not indexed under the exact term,
- or accessible via Isaac’s local environment rather than Hermes container.

### Required OpenClaw discovery tasks

1. Search past sessions broadly:
   ```text
   session_search queries:
   - OpenClaw
   - claw
   - OpenClaw memory
   - OpenClaw install
   - agent memory installation
   - Claude memory OpenClaw
   ```
2. Search local container paths:
   ```bash
   find /home/hermes -iname '*claw*' -o -iname '*open*claw*'
   ```
   Use Hermes file/search tools where possible.
3. Ask Isaac only if tool discovery fails, and ask specifically:
   - Where is the OpenClaw install located?
   - Is it on host `zicrone`, laptop, or inside the Hermes LXD container?
   - What part should be reused: memory schema, sync scripts, prompts, provider choice, UI, or agent architecture?
4. Once located, inspect its architecture before recommending anything:
   - persistence store,
   - retrieval mechanism,
   - write policy,
   - sync mechanism,
   - prompt injection strategy,
   - provider/API dependencies,
   - lessons/pitfalls.

## Research questions

### Hermes-native capabilities

Research and document:

- `hermes memory status`
- `hermes memory setup`
- supported providers and config shape
- provider plugin behavior
- how built-in memory is stored and injected
- how session_search indexes and retrieves
- whether `SYSTEM_BOOT.md` is truly auto-loaded or merely a documented convention
- how skills are selected and whether a boot skill/personality/soul file should be used
- how external memory provider output is injected into system/user context
- freshness, deletion, and privacy controls

### External provider candidates

Compare at least these, with hosted vs self-hosted options:

1. **Honcho** — likely preferred foundation for AI-native user modeling; investigate self-hosting.
2. **Mem0** — common hosted/local memory extraction and semantic recall.
3. **Supermemory** — hosted semantic memory option.
4. **Hindsight** — graph/entity-resolution style memory.
5. **RetainDB** — project/file/memory-type organization.
6. **OpenViking** — local/context database option.
7. **ByteRover** — developer/agent memory option.
8. **Plain markdown/Obsidian + git** — transparent human-readable memory layer.
9. **Vector DB options** if needed: Qdrant, Chroma, LanceDB, pgvector.
10. **Graph DB options** if needed: Neo4j, Kuzu, SQLite graph-like schema.

For each candidate, capture:

- hosted/self-hosted availability,
- license/cost,
- data ownership/privacy,
- API quality,
- local deployment complexity,
- memory model (facts, conversations, entities, embeddings, graph),
- deletion/retention support,
- integration path with Hermes,
- fresh-session recall test plan,
- failure modes.

## Desired architecture: layered memory, not one blob

Use separate layers for separate jobs:

| Layer | Purpose | Candidate implementation |
|---|---|---|
| Runtime config | Machine truth | `~/.hermes/config.yaml`, `.env`, `auth.json` |
| Boot context | Always-load critical state | `~/.hermes/SYSTEM_BOOT.md`, possibly a boot skill/personality |
| Durable compact facts | Stable preferences/environment facts | Hermes built-in memory |
| Conversation history | What was actually said/done | `session_search` / SQLite state DB |
| Procedural memory | Reusable workflows and pitfalls | Hermes skills |
| Human-readable knowledge | Decisions, operating manuals, project context | `hermes-memory` repo and/or Obsidian |
| Semantic memory | Associative recall/user modeling | Honcho or alternative provider |
| Reflection/maintenance | Keeps memory fresh | cron jobs, curator, review checklist |

## Write-routing policy to implement

After meaningful work, an agent should classify outputs:

1. **Config change** → update config + verification output.
2. **Critical current state** → update `SYSTEM_BOOT.md` and relevant policy doc.
3. **Reusable workflow/pitfall** → patch or create skill.
4. **Project/system knowledge** → update `hermes-memory` repo or Obsidian/project docs.
5. **Compact stable fact** → built-in memory.
6. **Historical detail only** → leave in session store; retrieve with `session_search`.
7. **Semantic association/user model** → external memory provider once configured.
8. **Temporary task progress** → do not persist as durable memory.

## Retrieval policy to implement

Before model/config/memory claims or changes, future agents should check:

1. Built-in memory already in prompt.
2. `~/.hermes/SYSTEM_BOOT.md`.
3. `~/.hermes/model-policy.md`.
4. Relevant skills, especially:
   - `hermes-agent`
   - `hermes-memory-providers`
   - `agent-brain-architecture`
   - `model-management`
5. Session search if the user says “last time,” “previously,” “the OpenClaw install,” etc.
6. Live system state (`hermes memory status`, `hermes config check`, file reads).
7. Web/source docs only after local state is inspected.

## Build phases

### Phase 0 — Baseline inventory

Deliverable: `memory-system-inventory.md`

- Current Hermes memory status.
- Current config/memory-related files.
- Current skills relevant to memory.
- Current `hermes-memory` repo structure.
- Session-search findings.
- OpenClaw install location/status.

### Phase 1 — Research and comparison

Deliverable: `memory-system-research.md`

- Compare Hermes built-in memory, Honcho, Mem0, Supermemory, Hindsight, RetainDB, OpenViking, ByteRover, markdown/Obsidian/git, and self-hosted vector/graph DB options.
- Include cost/privacy/deployment/integration tradeoffs.
- Prefer self-hosted when practical, but be honest about operational burden.

### Phase 2 — Architecture proposal

Deliverable: `memory-system-architecture.md`

- Recommended layered architecture.
- What gets stored where.
- Retrieval order.
- Write-routing policy.
- Fresh-session boot path.
- Security/privacy policy.
- Backup/export/restore story.
- Migration plan from current ad hoc files.

### Phase 3 — Prototype

Deliverable: working branch/config with test notes.

- Choose one provider/path to test first.
- Do not migrate everything.
- Use a tiny controlled set of test memories.
- Validate in a fresh session.

### Phase 4 — Verification and rollout

Deliverable: `memory-system-rollout.md`

- Fresh-session recall test.
- False-positive/stale-memory test.
- Deletion/update test.
- Cost/resource check.
- Gateway restart/reload behavior.
- Rollback plan.

### Phase 5 — Maintenance loop

Deliverable: cron/skill/checklist if useful.

- Weekly memory hygiene review.
- Stale assumption detector.
- Skill harvesting loop.
- SYSTEM_BOOT/model-policy drift checker.
- Optional sync/push job for `hermes-memory` repo.

## Acceptance criteria

The memory system is not “done” until a fresh Hermes session can correctly answer, with tool-backed verification:

1. What is the current model/fallback/OpenRouter policy?
2. Where is the canonical source of truth for model routing?
3. What should be saved to built-in memory vs skill vs model-policy vs session_search?
4. What was the OpenClaw installation and what lessons did we reuse or reject?
5. Which external memory provider was selected or deferred, and why?
6. How do we test that memory recall works and does not hallucinate stale state?

## Known pitfalls to avoid

- Do not dump every session outcome into built-in memory.
- Do not treat Honcho/Mem0/vector DB as a magic replacement for write policy.
- Do not store secrets in memory docs, skills, Obsidian, or repo files.
- Do not configure multiple external memory providers at once unless explicitly benchmarking.
- Do not assume hosted providers are acceptable for sensitive work; privacy and retention matter.
- Do not rely on stale browser snippets for provider capabilities; use docs/API/source.
- Do not forget that current memories may themselves be stale; verify live config.
- Do not implement before locating/reviewing the previous OpenClaw install or explicitly recording why it could not be found.

## Immediate next actions for the next agent

1. Load skills: `hermes-agent`, `hermes-memory-providers`, `agent-brain-architecture`, and likely `obsidian` if Obsidian is configured.
2. Run:
   ```bash
   hermes memory status
   hermes config check
   hermes status
   ```
3. Read:
   ```text
   ~/.hermes/SYSTEM_BOOT.md
   ~/.hermes/model-policy.md
   /home/hermes/claude-code-memory/README.md
   /home/hermes/claude-code-memory/context/current.md
   /home/hermes/projects/civicbridge/planning/handoffs/2026-06-07-memory-system-handoff.md
   ```
4. Use `session_search` to locate OpenClaw/openclaw/claw context.
5. If not found, ask Isaac a targeted question for the OpenClaw install location.
6. Produce Phase 0 inventory before proposing architecture.

## Related files

- Local copy of this handoff: `~/.hermes/planning/handoffs/2026-06-08-memory-system-buildout-handoff.md`
- Shared repo mirror: `/home/hermes/claude-code-memory/handoffs/2026-06-08-memory-system-buildout-handoff.md`
- Prior handoff: `/home/hermes/projects/civicbridge/planning/handoffs/2026-06-07-memory-system-handoff.md`
- Current model policy: `~/.hermes/model-policy.md`
- Boot context: `~/.hermes/SYSTEM_BOOT.md`
