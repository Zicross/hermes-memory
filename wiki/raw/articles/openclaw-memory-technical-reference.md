---
source_url: /mnt/openclaw/memory/resources/openclaw-memory-technical-reference.md
ingested: 2026-06-08
sha256: 24b98378083c74aa6e682d524f77d27357a0e50437f1ae0d099deb55a1aeb875
---

# OpenClaw Memory System — Technical Reference

> Generated: 2026-03-23 | Sources: OpenClaw official docs, GitHub issues, community guides

---

## 1. Architecture Overview

OpenClaw memory is **plain Markdown files on disk**. The files are the single source of truth — the model only "remembers" what gets written to disk. There is no magic RAM-based memory; conversation context is ephemeral and gets compacted away.

**Two persistence layers exist:**
- **Memory files** (Markdown in workspace) — durable, searchable, agent-controlled
- **Session transcripts** (JSONL) — append-only conversation logs, compacted over time

---

## 2. Bootstrap File Loading

### What Gets Injected Every Session
These workspace files are loaded into every session's system prompt:
- `AGENTS.md` — operating instructions
- `SOUL.md` — persona/tone
- `USER.md` — operator profile
- `IDENTITY.md` — agent name/emoji
- `TOOLS.md` — tool guidance (does NOT control tool availability)
- `HEARTBEAT.md` — optional heartbeat checklist
- `BOOT.md` — optional startup checklist
- `MEMORY.md` — curated long-term memory (**only in main/private sessions, never group**)

### Hard Limits
| Parameter | Default | Configurable? |
|---|---|---|
| **Per-file bootstrap max chars** | 20,000 | Yes: `agents.defaults.bootstrapMaxChars` |
| **Total bootstrap max chars** | 150,000 | Yes: `agents.defaults.bootstrapTotalMaxChars` |

- If a bootstrap file exceeds the per-file limit, it is **truncated**.
- If total bootstrap exceeds 150K chars, files are truncated/dropped.
- Missing files get a "missing file" marker injected (no crash).
- Both `MEMORY.md` and `memory.md` are loaded if both exist (deduplicated by realpath).

---

## 3. Memory File Layout

```
~/.openclaw/workspace/
├── MEMORY.md                    # Curated long-term memory (optional)
├── memory/
│   ├── YYYY-MM-DD.md           # Daily notes (one per day, append-only)
│   ├── projects/               # PARA: active work
│   ├── areas/                  # PARA: ongoing responsibilities
│   ├── resources/              # PARA: reference material
│   └── archive/                # PARA: completed/abandoned
└── skills/                     # Workspace-specific skills
```

**Key distinction:** Workspace (`~/.openclaw/workspace/`) is separate from state dir (`~/.openclaw/`) which holds config, credentials, sessions.

---

## 4. Memory Search (`memory_search` tool)

### What Gets Indexed
- **File types:** Markdown only (`.md` files)
- **Default scope:** `MEMORY.md` + `memory/**/*.md`
- **Extra paths:** Configurable via `memorySearch.extraPaths[]` (absolute or workspace-relative)
- **Symlinks:** Ignored (both files and directories)
- **Multimodal:** Images/audio from `extraPaths` only, requires Gemini embedding 2, opt-in

### Chunking
- **Target chunk size:** ~400 tokens
- **Chunk overlap:** ~80 tokens
- **Snippet return size:** ~700 chars max per result

### Index Storage
- Per-agent SQLite: `~/.openclaw/memory/<agentId>.sqlite`
- Configurable: `agents.defaults.memorySearch.store.path` (supports `{agentId}` token)
- sqlite-vec acceleration when available (falls back to JS cosine similarity)

### Freshness / Sync
- File watcher on `MEMORY.md` + `memory/` with **1.5s debounce**
- Sync runs: on session start, on search, or on interval (async)
- Session transcript indexing uses delta thresholds (async, best-effort)
- `memory_search` **never blocks** on indexing — results can be slightly stale

### Reindex Triggers
Index stores embedding provider/model + endpoint fingerprint + chunking params. If any change → full automatic reindex.

### Result Limits (configurable)
| Parameter | Default | Config Path |
|---|---|---|
| `maxResults` | 6 (QMD) | `memory.qmd.limits.maxResults` |
| `maxSnippetChars` | ~700 | `memory.qmd.limits.maxSnippetChars` |
| `maxInjectedChars` | — | `memory.qmd.limits.maxInjectedChars` |
| `timeoutMs` | 4000 (QMD) | `memory.qmd.limits.timeoutMs` |

### What `memory_search` Returns
- Snippet text (~700 chars)
- File path
- Line range
- Score
- Provider/model used
- Whether fallback was used
- Optional `Source: <path#line>` citation footer (when `memory.citations` = `auto`/`on`)

### What `memory_get` Does
- Reads a specific memory Markdown file (workspace-relative)
- Optional start line + line count
- Paths outside `MEMORY.md` / `memory/` are **rejected**
- Gracefully returns `{ text: "", path }` for missing files (no throw)

---

## 5. Embedding Providers

### Auto-Selection Priority (when `memorySearch.provider` not set)
1. `local` — if `memorySearch.local.modelPath` exists
2. `openai` — if OpenAI key resolvable
3. `gemini` — if Gemini key resolvable
4. `voyage` — if Voyage key resolvable
5. `mistral` — if Mistral key resolvable
6. **Disabled** if none available

Also supported: `ollama` (not auto-selected)

### Key Constraint
**Codex OAuth only covers chat/completions — it does NOT satisfy embedding API calls.** You need a separate API key for embeddings.

### Local Embedding
- Default model: `embeddinggemma-300m-qat-Q8_0.gguf` (~0.6 GB)
- Requires `pnpm approve-builds` + `pnpm rebuild node-llama-cpp`
- Auto-downloads GGUF on first use

### Fallback
- `memorySearch.fallback`: `openai | gemini | voyage | mistral | ollama | local | none`
- Used only when primary provider fails

---

## 6. Hybrid Search (BM25 + Vector)

### How It Works
1. Vector: top `maxResults × candidateMultiplier` by cosine similarity
2. BM25: top `maxResults × candidateMultiplier` by FTS5 rank
3. Union candidates, compute: `finalScore = vectorWeight × vectorScore + textWeight × textScore`

### Configuration
```json5
agents: {
  defaults: {
    memorySearch: {
      query: {
        hybrid: {
          enabled: true,
          vectorWeight: 0.7,        // default
          textWeight: 0.3,          // default
          candidateMultiplier: 4,   // default
          mmr: {
            enabled: false,         // default off
            lambda: 0.7             // 0=max diversity, 1=max relevance
          },
          temporalDecay: {
            enabled: false,         // default off
            halfLifeDays: 30        // score halves every 30 days
          }
        }
      }
    }
  }
}
```

### Temporal Decay Details
- Today's notes: 100% score
- 7 days: ~84%
- 30 days: 50%
- 90 days: 12.5%
- 180 days: ~1.6%
- **Evergreen files never decayed:** `MEMORY.md`, non-dated `memory/*.md`
- Dated files use filename date; others use mtime

### MMR (Diversity Re-ranking)
- Reduces near-duplicate results using Jaccard text similarity
- Off by default; enable when daily notes create redundant hits

---

## 7. QMD Backend (Experimental)

QMD = local-first search sidecar combining BM25 + vectors + reranking.

### Enabling
```json5
memory: {
  backend: "qmd",
  qmd: {
    includeDefaultMemory: true,
    update: { interval: "5m", debounceMs: 15000 },
    limits: { maxResults: 6, timeoutMs: 4000 },
    paths: [
      { name: "docs", path: "~/notes", pattern: "**/*.md" }
    ]
  }
}
```

### Key Properties
- Disabled by default; opt-in via `memory.backend = "qmd"`
- Requires separate `qmd` CLI install (Bun-based)
- Runs fully local via `node-llama-cpp`, auto-downloads GGUF models
- State lives under `~/.openclaw/agents/<agentId>/qmd/`
- Search modes: `search` (default), `vsearch`, `query`
- **Automatic fallback:** If QMD fails or binary missing → falls back to builtin SQLite
- Boot refresh runs in background by default (non-blocking)
- Update interval: default 5 minutes
- First search may be slow (model download)

### QMD Session Indexing
```json5
memory: {
  qmd: {
    sessions: {
      enabled: true,           // opt-in
      retentionDays: 30,
      exportDir: "..."
    }
  }
}
```
Exports sanitized User/Assistant turns into a dedicated QMD collection.

---

## 8. Session Management & Compaction

### Session Architecture
- Each channel/DM gets its own session (context does NOT carry between channels)
- Sessions stored: `~/.openclaw/agents/<agentId>/sessions/<sessionId>.jsonl`
- Session metadata: `~/.openclaw/agents/<agentId>/sessions/sessions.json`

### Compaction Triggers
1. **Overflow recovery:** model returns context overflow → compact → retry
2. **Threshold maintenance:** `contextTokens > contextWindow - reserveTokens`

### Compaction Settings
```json5
compaction: {
  enabled: true,
  reserveTokens: 16384,
  keepRecentTokens: 20000,
}
```

**Safety floor:** OpenClaw enforces minimum `reserveTokens` of **20,000 tokens** (configurable via `agents.defaults.compaction.reserveTokensFloor`, set to 0 to disable).

### Pre-Compaction Memory Flush
```json5
agents: {
  defaults: {
    compaction: {
      reserveTokensFloor: 20000,
      memoryFlush: {
        enabled: true,              // default: true
        softThresholdTokens: 4000,  // buffer before compaction threshold
        systemPrompt: "...",
        prompt: "..."
      }
    }
  }
}
```

**How it works:**
- Triggers when: `contextTokens > contextWindow - reserveTokensFloor - softThresholdTokens`
- Runs a silent agentic turn (NO_REPLY) that writes durable state to disk
- One flush per compaction cycle (tracked in `sessions.json`)
- Skipped if workspace is read-only
- Only runs for embedded Pi sessions (CLI backends skip it)

### Session Maintenance Defaults
| Parameter | Default |
|---|---|
| `pruneAfter` | 30 days |
| `maxEntries` | 500 |
| `rotateBytes` | 10 MB |
| `cron.sessionRetention` | 24 hours |

### Session Reset Triggers
- `/new` or `/reset` command
- Daily reset at 4:00 AM local time
- Idle expiry (`session.reset.idleMinutes`)
- Thread parent fork guard: `session.parentForkMaxTokens` = 100,000 (skips parent forking if too large)

---

## 9. Session Memory Search (Experimental)

```json5
agents: {
  defaults: {
    memorySearch: {
      experimental: { sessionMemory: true },
      sources: ["memory", "sessions"]
    }
  }
}
```

- **Opt-in**, off by default
- Indexed asynchronously with delta thresholds:
  - `deltaBytes`: 100,000 (~100 KB)
  - `deltaMessages`: 50 JSONL lines
- Results are snippets only; `memory_get` still limited to memory files
- Isolated per agent (only that agent's sessions indexed)
- Session logs on disk are readable by any process with filesystem access

---

## 10. Multi-Agent Memory

### Current State
- Each agent has its own workspace and memory index
- Memory search is **isolated per agent** — no built-in shared knowledge base
- Session indexing is per-agent only

### Sharing Workarounds
1. **`memorySearch.extraPaths`** — point multiple agents at a shared directory of `.md` files
2. **QMD `paths[]`** — add shared directories to QMD collections
3. **Symlinks** — NOT supported (ignored by indexer)
4. **Filesystem sharing** — agents can read/write to shared paths if they have OS-level access
5. **`sessions_send`** — agents can message each other, but this is conversation-based, not memory-based

### What CAN'T Be Done
- No native cross-agent memory search
- No shared vector index
- No memory federation or merge
- No cross-agent memory_get

---

## 11. Embedding Cache

```json5
memorySearch: {
  cache: {
    enabled: true,
    maxEntries: 50000
  }
}
```

Caches chunk embeddings in SQLite so reindexing doesn't re-embed unchanged text.

---

## 12. What CAN'T Be Changed vs What Can Be Tuned

### Hard Constraints (not configurable)
- Memory files must be Markdown (`.md`)
- `memory_get` only reads from `MEMORY.md` / `memory/` paths
- Symlinks are always ignored
- One flush per compaction cycle (no configurable frequency)
- Session transcripts are ephemeral (compaction is lossy)
- Codex OAuth doesn't work for embeddings
- Memory search never blocks on indexing

### Tunable Parameters (summary)
| Parameter | Config Path | Default |
|---|---|---|
| Bootstrap per-file max | `agents.defaults.bootstrapMaxChars` | 20,000 chars |
| Bootstrap total max | `agents.defaults.bootstrapTotalMaxChars` | 150,000 chars |
| Embedding provider | `memorySearch.provider` | auto-select |
| Hybrid search weights | `memorySearch.query.hybrid.vectorWeight/textWeight` | 0.7 / 0.3 |
| MMR diversity | `memorySearch.query.hybrid.mmr.enabled` | false |
| Temporal decay | `memorySearch.query.hybrid.temporalDecay.enabled` | false |
| Temporal half-life | `...temporalDecay.halfLifeDays` | 30 |
| QMD update interval | `memory.qmd.update.interval` | 5 min |
| QMD max results | `memory.qmd.limits.maxResults` | 6 |
| QMD timeout | `memory.qmd.limits.timeoutMs` | 4000 |
| Extra index paths | `memorySearch.extraPaths[]` | none |
| Session memory | `memorySearch.experimental.sessionMemory` | false |
| Compaction reserve | `compaction.reserveTokens` | 16,384 |
| Reserve floor | `compaction.reserveTokensFloor` | 20,000 |
| Memory flush enabled | `compaction.memoryFlush.enabled` | true |
| Flush soft threshold | `compaction.memoryFlush.softThresholdTokens` | 4,000 |
| Cache max entries | `memorySearch.cache.maxEntries` | 50,000 |
| Session prune after | `session.maintenance.pruneAfter` | 30 days |
| Max session entries | `session.maintenance.maxEntries` | 500 |
| File watcher debounce | — | 1.5s (hardcoded) |
| Chunk size | — | ~400 tokens (hardcoded) |
| Chunk overlap | — | ~80 tokens (hardcoded) |

---

## 13. Known Issues (GitHub)

- **#23409** — Agent workers can OOM a host (no memory limits/spawn caps)
- **#42877** — Feature request for bounded memory tool with hard character limits (2,200 chars for memory)
- **#18947** — OpenClaw requires >1GB RAM, limiting lightweight deployment
- **#11308** — QMD backend systemic issues: config complexity, poor error reporting, no health checks on startup

---

## Sources

- https://docs.openclaw.ai/reference/memory-config
- https://docs.openclaw.ai/reference/session-management-compaction
- https://docs.openclaw.ai/concepts/agent-workspace
- https://docs.openclaw.ai/concepts/memory
- https://github.com/openclaw/openclaw/issues/23409
- https://github.com/openclaw/openclaw/issues/42877
- https://github.com/openclaw/openclaw/issues/18947
- https://github.com/openclaw/openclaw/issues/11308
