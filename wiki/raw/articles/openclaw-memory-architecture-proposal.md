---
source_url: /mnt/openclaw/memory/projects/memory-architecture-proposal.md
ingested: 2026-06-08
sha256: 77cde01ed6e0d5c6c9041af8d3679f85e276ed8886b38517375426f94a223fc5
---

# Memory Architecture Proposal

**Created:** 2026-03-23
**Status:** Proposal — awaiting operator review
**Priority:** Critical (affects all future knowledge retention)

## The Problem

Current memory is a flat PARA structure with one growing file
(lessons-learned.md, 34 items) mixing permanent policies, temporal
lessons, operator directives, and technical facts. This will break in
three ways as it scales:

1. **Retrieval degradation** — QMD searches a 200-line file and can't
   distinguish a security policy from a one-time debugging note
2. **Decay confusion** — Nightly consolidation can't tell what's
   permanent vs. temporal, risks pruning critical rules
3. **Bootstrap bloat** — MEMORY.md (126 lines) is loaded into every
   session's context window. As it grows, it competes for tokens
   with actual conversation

## Design Principles

Drawn from three sources:

**AWS AgentCore pattern:** Extract → Consolidate → Retrieve. Memories
aren't just stored — they're classified by type (semantic facts,
preferences, summaries) and consolidated against existing knowledge
to prevent duplication and resolve conflicts.

**Zettelkasten + PARA hybrid:** PARA handles actionable items
(projects, areas). Zettelkasten handles permanent knowledge (atomic,
linked, never decays). The key insight: temporary project notes and
permanent knowledge need physical separation with functional linking.

**OpenClaw reality:** Memory files are plain markdown. MEMORY.md is
injected into every session. `memory/*.md` files are searchable via
QMD but NOT auto-loaded. This means MEMORY.md is prime real estate
(always visible) while memory/ files require explicit search.

## Proposed Architecture

### Tier 1: Always in Context (Bootstrap Files)

These are loaded every session. Must stay SMALL. Budget: ~5,000 chars
each, ~20,000 chars total for memory-related files.

```
MEMORY.md (workspace root)
├── Identity & Mission (who am I, what's the goal)
├── Active Registries (studios, channels — tables only)
├── Current Priorities (top 3-5 things that matter NOW)
└── Memory Navigation Guide (where to find what)
```

**MEMORY.md becomes an INDEX, not a store.** It tells me WHERE to
look, not WHAT to know. Everything else lives in searchable files.

### Tier 2: Permanent Knowledge Base (never decays)

These files are searched via QMD, never auto-loaded, never pruned
by nightly consolidation. They grow indefinitely but are organized
by domain so QMD can retrieve precisely.

```
memory/
├── kb/                          ← Knowledge Base (permanent)
│   ├── policies.md              ← Security, comms, spending rules
│   ├── operator-preferences.md  ← How Isaac wants things done
│   ├── system-reference.md      ← Technical facts, config gotchas
│   ├── business-context.md      ← Business model, revenue paths
│   └── agent-protocols.md       ← Inter-agent comms, delegation rules
```

**Rules for kb/ files:**
- Each item is atomic: one heading per fact/rule
- Every item has a date stamp (when learned)
- Items are grouped by domain, not by date
- NEVER pruned by nightly consolidation
- New items append; old items only removed if explicitly superseded
- Each file stays under 300 lines; split into sub-files when exceeded

### Tier 3: Active Work (temporal, managed by PARA)

```
memory/
├── projects/                    ← Active work with deadlines
│   ├── deep-research-skill.md
│   └── memory-architecture-proposal.md
├── areas/                       ← Ongoing responsibilities
│   └── lessons-learned.md       ← TEMPORAL only (see below)
├── resources/                   ← Reference material
│   └── browser-research-*.md
└── archive/                     ← Completed/abandoned work
```

### Tier 4: Daily Notes (ephemeral, decay-eligible)

```
memory/
├── 2026-03-23.md               ← Today's running log
├── 2026-03-22.md               ← Yesterday's log
└── (older notes summarized weekly, archived monthly)
```

## What Changes for lessons-learned.md

This file becomes TEMPORAL ONLY. It's the inbox for new lessons.
The nightly consolidation process must:

1. Review each item in lessons-learned.md
2. Classify: Is this a permanent rule, a preference, a tech fact,
   or a temporal observation?
3. Promote permanent items to the appropriate `kb/` file
4. Keep only unprocessed or genuinely temporal items in
   lessons-learned.md
5. Archive items older than 30 days that weren't promoted

**This means lessons-learned.md stays small** (only recent,
unprocessed items) while permanent knowledge accumulates in the
knowledge base.

## Migration Plan

### Phase 1: Create kb/ structure and migrate existing items

Take all 34 items from current lessons-learned.md and distribute:

| Current Item # | Classification | Destination |
|---------------|----------------|-------------|
| ClawHub policy | Security policy | kb/policies.md |
| sessions_send rule | Comms policy | kb/agent-protocols.md |
| Never kill without confirmation | Decision authority | kb/policies.md |
| Route heartbeat to #system | Operator preference | kb/operator-preferences.md |
| Quality > volume | Operator preference | kb/operator-preferences.md |
| Model names exact | System fact | kb/system-reference.md |
| Playwright needs libs | System fact | kb/system-reference.md |
| Revenue framing rule | Business rule | kb/business-context.md |
| ... (all 34 items classified) | ... | ... |

### Phase 2: Slim down MEMORY.md

Remove all content from MEMORY.md that doesn't need to be in every
session's context. Replace with an index:

```markdown
# Memory Index

## Mission
[2-3 sentence current mission statement]

## Active Studios
| Agent | Status | Domain |
|-------|--------|--------|

## Current Priorities
1. [Priority 1]
2. [Priority 2]
3. [Priority 3]

## Where to Find What
- Policies & rules → memory/kb/policies.md
- How Isaac wants things → memory/kb/operator-preferences.md
- Technical reference → memory/kb/system-reference.md
- Business context → memory/kb/business-context.md
- Agent protocols → memory/kb/agent-protocols.md
- Channel map → memory/kb/system-reference.md (channels section)
- Active projects → memory/projects/
- Daily notes → memory/YYYY-MM-DD.md
```

### Phase 3: Update nightly consolidation

Add a promotion step to the nightly routine:
1. Scan lessons-learned.md for items older than 24h
2. Classify each item
3. Append to appropriate kb/ file
4. Remove from lessons-learned.md

### Phase 4: Update AGENTS.md memory protocol

Add retrieval guidance:
- "Before answering about policies or rules → search kb/policies.md"
- "Before answering about operator preferences → search
  kb/operator-preferences.md"
- "When learning something new → write to lessons-learned.md first,
  nightly promotion handles the rest"

## Scaling Properties

At 1,000 items:
- MEMORY.md stays ~50 lines (index only)
- Each kb/ file has ~200 items, well within QMD search capability
- lessons-learned.md stays <20 items (recent only)

At 10,000 items:
- kb/ files split by sub-domain (e.g., kb/system/discord.md,
  kb/system/docker.md, kb/policies/security.md,
  kb/policies/spending.md)
- QMD handles file-level search efficiently
- MEMORY.md still ~50 lines

At 100,000 items:
- Consider adding a kb/index.md that maps topics to files
- QMD's hybrid search handles this volume natively
- May need embedInterval tuned down from 60m

## Why This Is Better

1. **MEMORY.md stays small forever** — it's an index, not a store
2. **Permanent knowledge never decays** — kb/ files are exempt from
   nightly pruning
3. **Fresh lessons have a clear path** — lessons-learned.md → nightly
   promotion → kb/ files
4. **QMD retrieval improves** — smaller, focused files mean more
   precise search results
5. **Scales indefinitely** — sub-file splitting handles growth
6. **Compatible with existing PARA** — projects/areas/resources/archive
   still work for temporal items
