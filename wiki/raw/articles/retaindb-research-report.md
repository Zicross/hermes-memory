# RetainDB Deep Research Report
*For Hermes Agent — Memory Provider Prototype Assessment*
*Researched: 2026-06-08 | Sources: README (SHA256: 339d699f), Hermes plugin source, web search*

---

## Executive Summary

RetainDB is a two-product memory infrastructure company: **RetainDB Local** (Apache 2.0, runs on your machine, no cloud account) and **RetainDB Server/Cloud** (BSL 1.1 server, managed hosted service). The Hermes plugin targets the **Cloud API** (`https://api.retaindb.com`) — it requires a paid API key and sends all data to RetainDB's cloud. **Local mode is NOT wired into the Hermes plugin** as of plugin v1.0.0. This is the single biggest finding for prototype planning.

**Verdict: RetainDB Cloud is a viable second prototype with strong retrieval mechanics and a well-designed plugin, but it is a paid cloud service, not a local-first option. RetainDB Local as a self-hosted backend is architecturally possible but requires manual bridging.**

---

## 1. Architecture

### RetainDB Local vs Server/Cloud

| Dimension | RetainDB Local | RetainDB Server | RetainDB Cloud |
|---|---|---|---|
| Start command | `npx -y @retaindb/local` | `docker compose up` / `pnpm dev:server` | Managed — no self-hosting |
| Port | `:3111` (API), `:3113` (viewer) | `:3000` | `api.retaindb.com` |
| Storage | Atomic disk snapshot + append-only journal under `~/.retaindb/` | Postgres + pgvector | Managed Postgres + vector infra |
| Embeddings | Hash-vector (default) or local-transformer | OpenAI / pgvector | OpenAI-backed, managed |
| Auth | None required for localhost | Optional `RETAINDB_API_KEY` shared key | Hosted API keys, team access |
| Dependencies | Node.js (npx) only — no Postgres, Redis, Kafka, Qdrant, Cloudflare | Postgres + pgvector required | None (managed) |
| License | Apache 2.0 | BSL 1.1 | Proprietary |
| Connectors | None | GitHub, Notion, Confluence, Slack, Discord, arXiv, npm/PyPI, PDF | Same |

### Local API (port 3111) Endpoints

**Standard API (shared with Server):**
- `POST /v1/memory` — store a memory
- `POST /v1/memory/bulk` — batch store
- `POST /v1/memory/search` — hybrid search
- `POST /v1/context/query` — packed context retrieval
- `POST /v1/context/pack` — token-budgeted coding context pack
- `POST /v1/context/delta` — changed context since prior pack
- `POST /v1/context/compress-output` — compress noisy tool output
- `POST /v1/context/code-map` — compact file/symbol map
- `POST /v1/memory/ingest/session` — ingest conversation messages
- `GET /v1/memory/session/:sessionId` — session memory list
- `GET /v1/memory/profile/:userId` — profile memory list
- `DELETE /v1/memory/:id` — delete/deactivate

**Local-only runtime endpoints:**
- `GET /retaindb/health` — health check
- `GET /retaindb/snapshot` — viewer snapshot data
- `GET /retaindb/graph` — concept graph (clickable in viewer)
- `GET /retaindb/replay/:sessionId` — session step-through replay
- `POST /retaindb/observe` — coding-agent lifecycle hook capture
- `POST /retaindb/consolidate` — trigger manual consolidation

### Viewer (port 3113)

The built-in viewer provides:
- Memory browser with filtering by type/project
- **Clickable concept graph** — shows memory relationships (updates, contradicts, supports, extends, derives)
- **Session replay** with step-through (tied to `/retaindb/replay/:sessionId`)
- Benchmark reports from `~/.retaindb/benchmarks/`
- Health and snapshot status

---

## 2. Memory Model

### Memory Types (12 total)

```typescript
type MemoryType =
  | "factual"         // Facts about the world or project
  | "preference"      // User style/workflow preferences
  | "semantic"        // Conceptual relationships
  | "procedural"      // How-to steps, workflows
  | "decision"        // Architectural/design decisions
  | "constraint"      // Hard limits, non-negotiables
  | "instruction"     // Persistent behavioral instructions
  | "goal"            // Objectives and desired outcomes
  | "event"           // Things that happened
  | "correction"      // Supersedes wrong/outdated info
  | "session_summary" // Rolled-up session digests
  | "project_state";  // Current project snapshot
```

*(The Hermes plugin's `retaindb_remember` tool exposes a subset: `factual`, `preference`, `goal`, `instruction`, `event`, `opinion`.)*

### Memory Fields

Each memory record includes:
- `content` — string content
- `memory_type` — one of the 12 types above
- `importance` — float 0–1
- `confidence` — confidence score
- `user_id`, `session_id`, `project` — scoping dimensions
- `embedding` — vector (optional, hash fallback)
- `timestamps` — created_at, last_accessed
- `access_count` — reinforcement counter (increment on retrieval)
- `strength` — computed from access_count + recency (used in decay)

### Graph Relationships (Server/Cloud)

```typescript
type RelationType =
  | "updates"      // This supersedes another memory
  | "extends"      // Adds detail to another
  | "contradicts"  // Conflicts with another
  | "supports"     // Corroborates another
  | "derives";     // Was inferred from another
```

**Temporal validity:** Server/Cloud deployments support `validFrom` and `validUntil` so stale facts can be superseded without deletion. A `correction` memory type sets `contradicts` edges on the old record.

### Concrete Examples

- `preference`: "User prefers concise responses with concrete next steps."
- `decision`: "The project standardizes on Postgres and pgvector for self-hosted search."
- `constraint`: "Local mode must work without cloud API keys."
- `correction`: "The old webhook path is deprecated; use `/v1/agent-events`."
- `procedural`: "Before release, run typecheck, build, benchmark, and smoke replay."

---

## 3. Retrieval Pipeline

### Hybrid Stack: BM25 + Vector + Graph → RRF → Reranker

The README specifies this pipeline concretely:

```
Query
  ├─ BM25 lexical search          → ranked list A (term overlap)
  ├─ Vector similarity search     → ranked list B (embedding cosine distance)
  └─ Graph signals                → ranked list C (relationship weights, access counts)
          │
          ▼
   RRF (Reciprocal Rank Fusion) — fuses A, B, C into single ranked list
          │
          ▼
   Reranker (cross-encoder or LLM-judged) — final relevance score
          │
          ▼
   top_k results returned (default: 8, max: 20)
```

**BM25:** Classic term-frequency/inverse-document-frequency lexical retrieval. Excellent for exact terminology (function names, error messages, library names) that vector search can miss.

**Vector search:** Semantic similarity via embeddings. Local mode uses hash-vector embeddings by default (fast, no model download). Optional local transformer embeddings via `retaindb install-embeddings` + `RETAINDB_EMBEDDING_PROVIDER=local-transformers`. Server/Cloud uses OpenAI embeddings.

**Graph signals:** Access count (reinforcement), last-access recency, relationship type weights. A memory that has been retrieved 10 times gets a higher graph score than one retrieved once.

**RRF (Reciprocal Rank Fusion):** Each result gets score = Σ 1/(k + rank_i) across all three lists (k is a stabilization constant, typically 60). Mathematically robust fusion that doesn't require normalizing scores across heterogeneous retrievers.

**Reranking:** Cross-encoder or LLM-judged final relevance. This is the expensive step — applied to the top-N (e.g., top-30) from RRF to produce final top-k for the context pack.

### Low-Signal Filtering

Before storage, content is filtered for signal quality. Noise (repetitive status messages, unchanged tool output, trivially short turns) is dropped at capture time, not at retrieval. This keeps the index clean.

---

## 4. Context Packs

### What They Are

A **context pack** is a token-budgeted, deduplicated, multi-source payload that an agent can inject into a prompt instead of manually collecting file contents, memory search results, and tool output.

**Request:**
```json
POST /v1/context/pack
{
  "query": "fix auth middleware tests",
  "cwd": ".",
  "files": ["src/auth.ts", "tests/auth.test.ts"],
  "token_budget": 1200
}
```

**Response includes:**
- `context` — packed memory + relevant file chunks + code map + compressed tool output
- `context_hash` — stable hash of this pack (used for delta tracking)
- `delta_context` — only changed parts when `previous_context_hash` is supplied
- `estimated_tokens` — token count of full context
- `estimated_delta_tokens` — token count of delta only
- `changed` — new/updated entries since prior pack
- `removed` — entries that dropped out of relevance

### Token Budget Mechanics

The budget (e.g., 1200 tokens) is divided across components:
1. **Relevant memories** — highest-scoring memories from hybrid retrieval
2. **File chunks** — relevant sections of specified files (not full files)
3. **Code map** — compact symbol/import map of the repo (not the full codebase)
4. **Compressed tool output** — last N terminal/test/build outputs, noise removed

The system trims lower-priority components first to stay within budget.

### Delta Compression

When an agent supplies `previous_context_hash`, the response returns only what changed since the last pack:
- New memories that became relevant
- File chunks that changed on disk
- Updated code map entries
- Changed tool output

This prevents agents from reprocessing the same context every turn — only deltas need re-embedding or re-reading.

### Tool Output Compression

`POST /v1/context/compress-output` applies targeted compression:
- **Keeps:** error messages, failing test names, stack traces, assertion failures
- **Drops:** passing test output, unchanged status lines, verbose progress bars, repeated log lines
- Result: agents see only actionable signal from build/test runs

### Code Map

`POST /v1/context/code-map` returns a compact symbol map — exported functions, classes, interfaces, imports — without full file contents. Agents use this to understand project structure in ~200 tokens vs. thousands.

---

## 5. Consolidation

Three processes run either via `retaindb consolidate` command or `POST /retaindb/consolidate`:

### Duplicate Cleanup
Finds semantically near-duplicate memories (cosine similarity above threshold, e.g. 0.95) and merges them, keeping the highest-importance, most-accessed version. The merged record gets the union of access counts and the higher confidence score.

### Session Rollups
After a session ends (or on explicit consolidation), individual turn-level memories are rolled up into a `session_summary` memory. The summary replaces the fine-grained events for long-term storage, freeing index space while preserving the gist.

### Stale Weak-Memory Decay
Memories with low `strength` (low importance × low access count × old last-access) are progressively deactivated rather than deleted. "Stale" = not accessed in N days and strength below threshold. Deactivated memories are excluded from retrieval but preserved for audit/replay. Active cleanup happens on `retaindb consolidate` runs.

**Memory reinforcement (inverse of decay):** Every time a memory is returned in search results, its `access_count` increments and `last_accessed` updates. This raises its strength, protecting it from future decay.

---

## 6. Hermes Integration

### Plugin Location
```
/home/hermes/.local/share/pipx/venvs/hermes-agent/lib/python3.12/site-packages/
  plugins/memory/retaindb/__init__.py    (32,799 bytes — full implementation)
  plugins/memory/retaindb/plugin.yaml   (180 bytes — manifest)
```

### Plugin Configuration
```yaml
name: retaindb
version: 1.0.0
description: "RetainDB — cloud memory API with hybrid search and 7 memory types."
pip_dependencies:
  - requests
requires_env:
  - RETAINDB_API_KEY
```

**⚠️ Critical: The plugin hardcodes `_DEFAULT_BASE_URL = "https://api.retaindb.com"`. It targets the cloud API, not localhost:3111. RETAINDB_BASE_URL can override this, but the plugin.yaml marks RETAINDB_API_KEY as required — no local-mode bypass exists.**

### Plugin Architecture

#### Class: `RetainDBMemoryProvider(MemoryProvider)`

**Implements the full `MemoryProvider` interface:**

| Method | Purpose |
|---|---|
| `initialize(session_id, **kwargs)` | Sets up `_Client`, `_WriteQueue`, seeds SOUL.md to cloud |
| `system_prompt_block()` | Injects "RetainDB Memory Active. Project: X." into system prompt |
| `queue_prefetch(query)` | Fires 3 background threads at turn-end |
| `prefetch(query)` | Consumes prefetch results at next turn-start |
| `sync_turn(user, assistant)` | Queues turn for async cloud ingest |
| `get_tool_schemas()` | Returns 10 tool schemas |
| `handle_tool_call(name, args)` | Dispatches tool calls |
| `on_memory_write(action, target, content)` | Mirrors built-in memory writes to RetainDB |
| `shutdown()` | Drains prefetch threads + write queue |

#### Background Prefetch (3 threads, fire at turn-end, consumed at turn-start)

1. **`retaindb-ctx`**: Calls `/v1/context/query` + `/v1/memory/profile/:userId` → builds overlay block
2. **`retaindb-dialectic`**: Calls `/v1/memory/profile/:userId/ask` with LLM-powered dialectic synthesis (reasoning level: low/medium/high based on query length)
3. **`retaindb-agent-model`**: Calls `/v1/memory/agent/:agentId/model` → loads persona, persistent instructions, working style

The prefetch results are assembled into the `prefetch()` return value:
```
[RetainDB Context]
Profile:
- <top 5 profile memories>
Relevant memories:
- <top 5 context query results>

[RetainDB User Synthesis]
<dialectic LLM synthesis>

[RetainDB Agent Self-Model]
Persona: <agent persona>
Instructions:
- <instruction 1>
Working style: <style>
```

#### Durable Write-Behind Queue (`_WriteQueue`)

- Backed by **SQLite** at `~/.hermes/retaindb_queue.db`
- Crash-safe: pending rows survive restarts and replay on next startup
- Turn messages are enqueued immediately; a background daemon thread calls `/v1/memory/ingest/session` asynchronously
- On failure: row stays in SQLite with `last_error` field, retried after 2s
- At most 200 pending rows replayed on startup

#### SOUL.md Seeding

On `initialize()`, if `~/.hermes/SOUL.md` exists, its content is sent to `/v1/memory/agent/hermes/seed` in background. This seeds Hermes's agent identity (persona, instructions, working style) into RetainDB's agent model.

#### Project Naming Logic

```python
# Priority: RETAINDB_PROJECT env > hermes-<profile_name> > "default"
if explicit RETAINDB_PROJECT:
    project = explicit
elif profile_name (from hermes_home basename):
    project = f"hermes-{profile_name}"
else:
    project = "default"
```

### 10 Exposed Tools

| Tool | API Call | Purpose |
|---|---|---|
| `retaindb_profile` | `GET /v1/memory/profile/:userId` | Full stable profile |
| `retaindb_search` | `POST /v1/memory/search` | Semantic search (top_k 1–20) |
| `retaindb_context` | `POST /v1/context/query` + profile | Synthesized context block |
| `retaindb_remember` | `POST /v1/memory` | Save explicit memory |
| `retaindb_forget` | `DELETE /v1/memory/:id` | Delete memory |
| `retaindb_upload_file` | `POST /v1/files` (multipart) | Upload file, get rdb:// URI |
| `retaindb_list_files` | `GET /v1/files` | List file store |
| `retaindb_read_file` | `GET /v1/files/:id/content` | Read text file content |
| `retaindb_ingest_file` | `POST /v1/files/:id/ingest` | Extract memories from file |
| `retaindb_delete_file` | `DELETE /v1/files/:id` | Delete file |

#### What Data Is Sent per Turn

Every turn that has user+assistant content is enqueued with:
```json
{
  "project": "<project>",
  "session_id": "<session_id>",
  "user_id": "<user_id>",
  "messages": [
    {"role": "user", "content": "<full user turn>", "timestamp": "..."},
    {"role": "assistant", "content": "<full assistant turn>", "timestamp": "..."}
  ],
  "write_mode": "sync"
}
```

**This means full conversation content goes to `api.retaindb.com` on every turn.**

---

## 7. Benchmark Verification

### Numbers Claimed in Task Brief
- LoCoMo: 96.1%
- LongMemEval-S: 92.8%

### What RetainDB Actually Claims

**From RetainDB's own benchmark page (`retaindb.com/benchmark`, March 2026):**
- LongMemEval (ICLR 2025): **88% preference recall**, **79% overall**
- Hallucination rate: 0%

**From `retaindb.com/vs/mem0` comparison page:**
- LongMemEval overall score: 79%

**The 96.1% LoCoMo / 92.8% LongMemEval-S numbers belong to ByteRover, not RetainDB.**

Evidence trail:
1. `github.com/campfirein/byterover-cli` — "LoCoMo - 96.1% overall accuracy (1,982 questions, 272 docs). LongMemEval-S - 92.8% overall accuracy (500 questions, 23,867 docs)."
2. arxiv.org/html/2604.01599v1 — ByteRover paper "Agent-Native Memory Through LLM-Curated Hierarchical Context"
3. ByteRover blog: "ByteRover 2.1.5 scores 92.8% on LongMemEval-S - the highest accuracy in the market"

**Conclusion:** The benchmark numbers in the task brief were misattributed. RetainDB's verified published numbers are 79% overall / 88% preference recall on LongMemEval. ByteRover holds the 96.1% / 92.8% records. Both are Hermes memory provider competitors — RetainDB and ByteRover are separate products.

For context, Mem0 v3 (April 2026) claims 94.8% on LongMemEval and 91.6% on LoCoMo.

---

## 8. Self-Host Setup

### Node.js Requirement

The container has **Node.js v24.16.0 / npm 11.13.0** — this meets any reasonable requirement (RetainDB Local is an npm package; no Node version constraint is documented in the README beyond "npx").

### One-Command Start

```bash
npx -y @retaindb/local          # Start API on :3111, viewer on :3113
npx -y @retaindb/local demo     # Seed demo memories and run searches
npx -y @retaindb/local mcp      # Run MCP bridge (requires RETAINDB_BASE_URL)
npx -y @retaindb/local connect all   # Write agent config snippets (Codex, Claude Code, OpenCode)
```

### Disk Layout Under `~/.retaindb/`

From the README and configuration table:
```
~/.retaindb/
  local-store.json          # Atomic disk snapshot (RETAINDB_STORE)
  local-store.journal       # Append-only journal (crash recovery)
  benchmarks/               # Local benchmark reports from `retaindb benchmark`
  [embeddings/]             # Optional: local transformer model weights (install-embeddings)
```

No `~/.retaindb/` directory exists yet on this machine — it will be created on first `npx -y @retaindb/local` run.

### Self-Hosted Server (Full Stack)

```bash
git clone https://github.com/retaindb/retaindb
cd retaindb
docker compose up
# OR
cp .env.example packages/server/.env
pnpm install
pnpm --filter @retaindb/server run db:push   # Postgres + pgvector required
pnpm dev:server                              # Runs on :3000
```

Required env vars for server:
- `DATABASE_URL` — Postgres connection string (pgvector extension required)
- `RETAINDB_API_KEY` — optional shared deployment key
- `OPENAI_API_KEY` — optional, enables OpenAI embeddings
- `EXTRACTION_MODEL` — defaults to `gpt-4o-mini`

### Useful CLI Commands

```bash
retaindb                  # start local memory
retaindb benchmark        # run recall/latency benchmark
retaindb import-jsonl     # import Claude-style JSONL transcripts
retaindb consolidate      # dedupe and roll up memories
retaindb reembed          # refresh embeddings
retaindb doctor           # print local status
retaindb install-embeddings  # download local transformer model
```

---

## 9. Privacy

### Local Mode (RetainDB Local)

**Everything stays on disk, nothing leaves the machine:**
- Storage: `~/.retaindb/local-store.json` + journal — plain filesystem
- No cloud account, no API key, no network requests
- Embeddings: hash-vector (default, computed locally) OR local transformer model (optional install)
- No telemetry mentioned in README
- Code maps, file chunks, tool output — all local

**Local mode is fully air-gapped.**

### Hermes Plugin Mode (Cloud API)

**ALL conversation data goes to `api.retaindb.com`:**
- Every user+assistant turn is sent to `/v1/memory/ingest/session`
- Profile queries hit the cloud on every turn-start (prefetch)
- SOUL.md content is sent to the cloud on initialization
- Files uploaded via `retaindb_upload_file` go to cloud storage
- The `retaindb_queue.db` SQLite file is a local buffer only — it drains to the cloud

**Privacy exposure with cloud plugin:** High. Full conversation text, user identity, session IDs, file contents, agent persona — all cloud-stored.

### Self-Hosted Server Mode

Point `RETAINDB_BASE_URL` at your own server (`http://localhost:3000`) and data stays on your hardware. The server is BSL 1.1 licensed (free to self-host; not free to operate as a hosted service for others).

### Summary

| Mode | Data Stays Local? |
|---|---|
| RetainDB Local (`npx @retaindb/local`) | ✅ 100% local |
| Hermes plugin → Cloud API | ❌ All data to retaindb.com |
| Hermes plugin → Self-hosted server | ✅ if you run the server |
| Hermes plugin → localhost:3111 (Local) | ⚠️ Possible but not wired by default |

---

## 10. Limitations and Comparison to Honcho

### RetainDB Limitations

1. **Hermes plugin requires cloud API key** — `requires_env: RETAINDB_API_KEY` is hard-required. No local-mode path in the plugin. To use RetainDB Local as the backend, you'd need to set `RETAINDB_BASE_URL=http://localhost:3111` and remove the API key requirement (or add a dummy key — unclear if Local validates it).

2. **Paid tier only for cloud** — RetainDB Cloud has no documented free tier (contrast: ByteRover has a free local mode; Mem0 has a freemium plan). Vectorize.io comparison describes RetainDB as "Paid" vs. ByteRover as "Freemium".

3. **Server is BSL 1.1** — The self-hosted server cannot be used to build a hosted service without a commercial license. Fine for personal/team use.

4. **No structured user/agent modeling API** — The Hermes plugin calls `/v1/memory/profile/:userId/ask` (dialectic synthesis) and `/v1/memory/agent/:agentId/model`, but these are RetainDB-specific extensions, not a formal user-modeling framework.

5. **Benchmark numbers are lower than task brief implied** — RetainDB's own published numbers (79% LongMemEval overall) are respectable but below Mem0 v3 (94.8%) and ByteRover (92.8%).

6. **Single-tenant OSS server** — The self-hosted server doesn't support multi-tenant isolation natively. Cloud-style org isolation belongs in RetainDB Cloud only.

7. **Local embeddings are hash-based by default** — Hash-vector embeddings have lower semantic quality than real embeddings. Local transformer embeddings require an optional install step and a native runtime.

8. **Code map / context pack features are Local-specific** — The coding-agent context router (`/v1/context/pack`, `/v1/context/code-map`, delta compression) are Local runtime features and are NOT exposed through the Hermes plugin's tool schemas. The plugin only covers the memory CRUD + file store.

### Comparison to Honcho

| Dimension | RetainDB | Honcho |
|---|---|---|
| **Core metaphor** | Memory store + retrieval pipeline | User/agent relationship modeling |
| **Data model** | 12 typed memories, graph edges | Users, agents, sessions, dialectic sessions |
| **Retrieval** | BM25 + vector + graph + RRF + reranker | Semantic search + LLM-curated user model |
| **User modeling** | Profile endpoint + dialectic synthesis (add-on) | First-class `user_model` / `agent_model` |
| **Local option** | Yes (RetainDB Local, Apache 2.0) — but not wired to Hermes plugin | No local mode — cloud only |
| **Pricing** | Paid cloud; free Local; BSL self-hosted | Varies (API pricing) |
| **Coding-agent features** | Context packs, delta compression, code maps, session replay, MCP | Not coding-specific |
| **Framework adapters** | Vercel AI SDK, LangChain, LangGraph | More general |
| **Connectors** | GitHub, Notion, Confluence, Slack, Discord, arXiv, npm, PDF | Not documented |
| **Benchmark** | 79% LongMemEval overall | Not published |
| **Hermes plugin maturity** | Full-featured (10 tools, prefetch, write queue) | Also integrated |

**Key architectural difference:** Honcho's thesis is *modeling the relationship between user and agent over time* — it maintains explicit user models, agent models, and their evolution. RetainDB's thesis is *structured memory with high-quality retrieval* — it stores and retrieves typed facts with hybrid search. Honcho is better for "who is this user and how do they interact with agents"; RetainDB is better for "what did this agent learn that it should remember."

---

## 11. Readiness Assessment for Hermes Prototype

### What Works Today (Cloud Mode)

- Plugin is installed and functional in hermes-agent v0.15.2
- Activation requires: `RETAINDB_API_KEY=<key>` environment variable
- Project is auto-named `hermes-default` (default profile) or `hermes-<profile>`
- All 10 tools available immediately on activation
- 3-thread prefetch (context + dialectic + agent-model) fires automatically
- Write queue persists turns crash-safely to `~/.hermes/retaindb_queue.db`
- SOUL.md seeding happens on first `initialize()`

### Gap: Local Mode Bridge

To use RetainDB Local as the backend (avoiding cloud costs and privacy exposure):
```bash
npx -y @retaindb/local   # Start local server on :3111
export RETAINDB_BASE_URL=http://localhost:3111
export RETAINDB_API_KEY=local-dev-key  # Local may not validate this
```
**Unknown:** Whether RetainDB Local validates the API key. If it accepts any non-empty string, this would work. The README says "Local development can run open on localhost" — suggesting no key enforcement.

### Prototype Recommendation

**RetainDB Cloud** is a strong second prototype candidate because:
1. Plugin is fully written, installed, and battle-tested (10 tools, prefetch, write queue)
2. Dialectic synthesis and agent-model seeding are unique features not in Honcho
3. File store (upload/ingest/read/delete) enables document-as-memory workflows
4. Context pack + delta compression features (even if not in the Hermes plugin) are valuable for future coding-agent integration
5. The retrieval pipeline (BM25 + vector + graph + RRF + reranker) is the most complete hybrid stack of any current Hermes plugin

**Risks:**
- Cost: all turns go to paid cloud API
- Privacy: full conversation content sent to `api.retaindb.com`
- Dependency: relies on external service availability

**Recommended prototype path:** Start with Cloud API for capability exploration, then explore `RETAINDB_BASE_URL=http://localhost:3111` override with the Local runtime for a privacy-safe local variant.

---

## Files Created

- `/home/hermes/claude-code-memory/wiki/raw/articles/retaindb-research-report.md` — this report

## Sources Used

1. `/home/hermes/claude-code-memory/wiki/raw/articles/retaindb-readme.md` — full README (SHA256: 339d699f)
2. `/home/hermes/.local/share/pipx/venvs/hermes-agent/lib/python3.12/site-packages/plugins/memory/retaindb/__init__.py` — 766-line plugin source
3. `/home/hermes/.local/share/pipx/venvs/hermes-agent/lib/python3.12/site-packages/plugins/memory/retaindb/plugin.yaml` — plugin manifest
4. Web search: RetainDB benchmark page, ByteRover paper, Mem0 comparison, Vectorize comparison, agent memory 2026 guides
5. Live system checks: Node.js v24.16.0 confirmed; `~/.retaindb/` not yet created
