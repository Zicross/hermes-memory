---
source_url: /mnt/openclaw/memory/archive/research-2026-03/memory-architecture-final-report-2026-03-24.md
ingested: 2026-06-08
sha256: d1e01573a0ad3b9f9e97152d5c009ffd4617565c89bfda7d9cf37c4285f9c8b7
---

# Memory Architecture: Final Research Report & Recommendation

**Date:** 2026-03-24
**Requested by:** Isaac (overnight task from 2026-03-23)
**Confidence:** High — synthesized from 6+ sources, official docs, and proven implementations

---

## Executive Summary

Our current memory system works but has three problems that will compound:
1. **MEMORY.md is already at 126 lines / 7.7KB** — VelvetShark says keep it under 100 lines
2. **No separation between searchable knowledge and always-loaded context** — everything competes for bootstrap space
3. **No structured knowledge base** — lessons, decisions, and facts live in flat files without relationships

The recommended fix is a **3-layer hybrid** modeled on Felix's system (proven at $100K+ revenue) adapted to our multi-agent setup, implemented using OpenClaw's built-in capabilities (no external dependencies).

---

## What We Learned

### Source 1: Felix / Nat Eliason ($14K in 3 weeks, $100K+ total)

Felix uses a 3-layer memory system that Nat calls "the single biggest unlock":

| Layer | What | Where | When Accessed |
|-------|------|-------|---------------|
| Knowledge Graph | Durable facts about people, projects, business | `~/life/` using PARA | On-demand via search |
| Daily Notes | What's happening right now, running log | `memory/YYYY-MM-DD.md` | Auto-loaded (today + yesterday) |
| Tacit Knowledge | Preferences, habits, rules, lessons | Bootstrap files | Every session start |

**Key quote:** "Get the memory structure in first because then your conversations from day one will be useful."

**How Felix's consolidation works:**
- Bot writes to daily notes during conversations
- Nightly consolidation extracts important stuff from daily notes → knowledge graph
- Summary files within the knowledge graph enable quick lookups
- Git backup multiple times per day

Source: creatoreconomy.so, every.to

### Source 2: VelvetShark (OpenClaw Core Maintainer)

The Memory Masterclass is the most technically authoritative source. Key findings:

**Technical constraints we must design around:**
- Per-file bootstrap limit: **20,000 chars** (~5K tokens)
- Total bootstrap limit: **150,000 chars** (~50K tokens)
- Sub-agents only receive AGENTS.md + TOOLS.md — NOT SOUL.md, MEMORY.md, USER.md
- Daily logs are NOT bootstrap-injected — they're accessed via memory_search/memory_get
- memory_search indexes MEMORY.md + everything in `memory/` directory

**The memory protocol (proven pattern):**
```
Before answering about past work → search memory first
Before starting any task → check today's daily note
When corrected → add rule to MEMORY.md immediately
When session ending → summarize to daily log
```

**Maintenance cadence:**
- Daily: append to daily log (automatic via memory flush)
- Weekly: promote durable rules from daily logs → MEMORY.md
- Keep MEMORY.md short — anything not needed every session goes in daily logs

**Config recommendations (already partially applied):**
- reserveTokensFloor: 40,000 ✅ (applied 2026-03-23)
- contextPruning: cache-ttl mode ✅ (already configured, we use 30m TTL)
- memoryFlush: enabled ✅ (already configured)

Source: velvetshark.com/openclaw-memory-masterclass

### Source 3: Official OpenClaw Docs

**Memory is plain Markdown in the workspace. Period.**
- Files are the source of truth
- The model only "remembers" what gets written to disk
- Two default layers: `MEMORY.md` (curated) + `memory/YYYY-MM-DD.md` (daily)
- Today + yesterday's daily notes are auto-read at session start
- `memory_get` degrades gracefully when files don't exist

**Vector search supports:**
- Multiple embedding providers (we use QMD with BM25 + vector hybrid)
- MMR diversity re-ranking ✅ (enabled)
- Temporal decay ✅ (enabled, 30-day half-life)

Source: docs.openclaw.ai/concepts/memory

### Source 4: AI Memory Framework Research (Mem0, LangChain, CrewAI, A-MEM)

**Scaling thresholds:**
| Items | What Works | What Breaks |
|-------|-----------|-------------|
| <100 | Flat files | Nothing yet |
| 100-1,000 | Vector search needed | Flat files become junk drawers |
| 1,000-10,000 | Managed retrieval, metadata filtering | Simple vector returns noise |
| 10,000+ | Hybrid graph+vector | Everything else |

**We're currently at ~750 lines across memory files** — approaching the zone where pure flat files start degrading. The QMD hybrid search buys us time, but structure matters more as we scale.

**Key patterns from production frameworks:**
- Mem0: auto-capture + auto-recall, compaction-immune, but requires external service
- CrewAI: hierarchical scopes (/project/alpha, /agent/researcher) — useful for multi-agent
- A-MEM (Zettelkasten-inspired): atomic notes with auto-linking, memory evolution

Source: arxiv.org (Mem0 paper), various framework docs

### Source 5: Community Alternative Plugins

| Plugin | What It Solves | Complexity | Status |
|--------|---------------|------------|--------|
| QMD | Better retrieval (BM25 + vector) | Low | ✅ Already deployed |
| Mem0 | Compaction immunity | Medium-High | Consider later |
| Cognee/Graphiti | Structured relationships | High | Future (10K+ items) |
| Obsidian integration | User-visible memory editing | Low | Nice-to-have |

Source: github.com/openclaw, mem0.ai, community discussions

---

## Current State Audit

### What's working:
- ✅ QMD hybrid search (BM25 + vector + re-ranking)
- ✅ Session indexing (30-day retention)
- ✅ Memory flush before compaction
- ✅ reserveTokensFloor at 40K
- ✅ Memory protocol in AGENTS.md (search before acting)
- ✅ Daily notes system
- ✅ Nightly consolidation cron

### What needs fixing:
- ❌ MEMORY.md is 126 lines / 7.7KB — over the recommended 100-line limit
- ❌ No separation between "always in context" cheat sheet and "searchable knowledge"
- ❌ lessons-learned.md is 276 lines and growing — becoming a junk drawer
- ❌ No structured knowledge files (people, projects, decisions)
- ❌ No weekly promotion cycle from daily logs → structured files
- ❌ Multi-agent memory is isolated — Big J and Forge can't access Z's knowledge

---

## Recommendation: 3-Layer Hybrid Architecture

### Layer 1: Bootstrap Files (always in context — 20K char budget each)

These files are loaded every session. They're prime real estate. Keep them focused.

| File | Purpose | Target Size |
|------|---------|-------------|
| MEMORY.md | **Cheat sheet only** — top 50-80 lines of critical active context | <4KB |
| SOUL.md | Identity, personality, communication style | Current (stable) |
| AGENTS.md | Operating procedures, memory protocol, decision rules | Current (stable) |
| USER.md | Operator profile, preferences, environment | Current (stable) |
| TOOLS.md | Tool notes, environment details | Current (stable) |
| IDENTITY.md | Name, emoji, creature, vibe | Current (stable) |
| HEARTBEAT.md | Heartbeat checklist | Current (stable) |

**Action: Slim MEMORY.md from 126 lines → ~80 lines.** Move searchable content to the knowledge base.

### Layer 2: Knowledge Base (searchable via memory_search — no bootstrap cost)

This is the new layer. Structured markdown files in `memory/` that QMD indexes automatically.

```
memory/
├── YYYY-MM-DD.md          # Daily notes (existing, keep as-is)
├── areas/
│   ├── lessons-learned.md  # Rules from mistakes (existing, restructure)
│   ├── operator-preferences.md  # What Isaac likes/dislikes (NEW)
│   └── system-knowledge.md     # OpenClaw config patterns, tech decisions (NEW)
├── projects/
│   ├── <project-slug>.md   # Active project context (existing pattern)
│   └── ...
├── resources/
│   ├── people.md           # Key people, contacts, relationships (NEW)
│   ├── business-decisions.md   # Decision log with reasoning (NEW)
│   └── revenue-paths.md       # Business model analysis (NEW)
└── archive/
    └── ...                 # Completed/abandoned work
```

**Key design decisions:**
- Files live in `memory/` so QMD indexes them automatically
- PARA structure maintained but flattened (no deep nesting)
- Each file has a clear single responsibility
- Files can grow large — that's fine because they're searchable, not bootstrap-loaded
- Agent sub-directories (memory/projects/amazon/, memory/projects/forge/) for agent-scoped knowledge

### Layer 3: Tacit Knowledge (embedded in bootstrap files)

Felix's insight: tacit knowledge is different from rules or facts. It's "how you work" — communication patterns, decision styles, workflow habits.

For us, this lives in:
- **SOUL.md** — how Big Z communicates and decides
- **USER.md** — how Isaac communicates and decides
- **AGENTS.md** — operational rules, anti-patterns

No structural change needed here — we already have this layer. Just need to be intentional about what goes where.

### Maintenance Schedule

| Cadence | Action | Owner |
|---------|--------|-------|
| Continuous | Write to daily log during conversations | Memory flush + agent |
| Nightly (2 AM ET) | Consolidation: daily logs → structured knowledge files | Nightly cron |
| Weekly (Sunday 3 AM ET) | Promotion: review knowledge files, slim MEMORY.md, archive old daily logs | Weekly cron (NEW) |
| Monthly | Review archive, delete low-value items | Manual or cron |

### Multi-Agent Memory Sharing

Current problem: Big J and Forge can't access Z's knowledge base.

**Proposed solution (Level B — needs operator approval):**
1. Shared knowledge lives in Z's workspace (`memory/resources/`)
2. Studio CEOs that need cross-agent context use `sessions_send` to ask Z
3. Z searches his memory and returns relevant context
4. No direct file sharing between agent workspaces (security boundary)

**Future consideration:** If agents need direct memory access, we could:
- Symlink shared files into each agent's workspace
- Use QMD's `paths` config to index across workspaces
- But this breaks isolation — only do it with explicit need

---

## Implementation Plan

### Phase 1: Restructure (this week)
1. Create new knowledge base files in `memory/resources/`
2. Slim MEMORY.md: move searchable content to knowledge base
3. Restructure lessons-learned.md: categorize by domain
4. Verify QMD indexes the new files

### Phase 2: Automate (next week)
1. Create weekly consolidation cron (Sunday 3 AM ET)
2. Update nightly consolidation to promote to knowledge base files (not just daily logs)
3. Add git backup of workspace

### Phase 3: Evaluate (week 3+)
1. Monitor memory search quality after restructure
2. Track MEMORY.md size — should stay under 80 lines
3. Assess if multi-agent sharing needs more than sessions_send
4. Evaluate Mem0 if compaction becomes a real problem

---

## Cost Analysis

| Item | Cost | Notes |
|------|------|-------|
| Restructure | 0 | File moves only |
| Weekly cron | ~$0.02/week | Haiku-class model, light context |
| Git backup | 0 | Local git, no remote needed |
| Mem0 (if needed later) | Self-hosted free, or $99/mo cloud | Only if compaction becomes a real problem |

---

## Risk Assessment

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| MEMORY.md too aggressive slim → agent loses context | Medium | Low (reversible) | Keep backup, test gradually |
| QMD doesn't index new files | Low | Medium | Verify after restructure |
| Weekly cron misses important promotions | Medium | Low | Nightly consolidation is safety net |
| Multi-agent isolation causes knowledge silos | High | Medium | Z as knowledge broker via sessions_send |

---

## Appendix: What NOT To Do

Based on research, these approaches are anti-patterns:

1. **Don't use MEMORY.md as a journal** — it's a cheat sheet, not a log
2. **Don't store everything in bootstrap files** — you'll hit the 150K char total limit
3. **Don't create deeply nested folder structures** — QMD search works on flat patterns
4. **Don't rely on conversation history for durable knowledge** — compaction will eat it
5. **Don't install Mem0/external memory until file-based memory fails** — premature complexity
6. **Don't share workspaces between agents** — breaks isolation, creates confusion
7. **Don't skip the weekly promotion cycle** — daily logs pile up and become unsearchable noise
