# Honcho Deep Research Report
**Date:** 2026-06-08  
**Purpose:** Evaluate Honcho as the first external semantic memory provider prototype for Hermes Agent  
**Sources:** Primary source code (`/home/hermes/.local/share/pipx/venvs/hermes-agent/lib/python3.12/site-packages/plugins/memory/honcho/`), official README (ingested at wiki), honcho.dev docs, Plastic Labs blog

---

## 1. Architecture: How Honcho Stores Messages, Builds Peer Representations, and Query Interface

### Two-Service Split
Honcho is architecturally split into:

- **Storage Service**: Synchronous, handles workspaces → peers → sessions → messages via REST API. This is the source of truth.
- **Insights Service**: Asynchronous, runs background reasoning ("deriver" worker process) that consumes the storage layer and builds per-peer representations, conclusions, summaries, and peer cards.

### Storage Pipeline
```
User message → session.add_messages() → Honcho DB (Postgres + pgvector)
                                         ↓
                               Derivation task enqueued
                                         ↓
                          Deriver worker processes background:
                            - representation: update peer model
                            - summary: session summarization
                            - dream: specialized insights processing
                                         ↓
                          Internal collections (observer, observed) keyed by peer pairs
                                         ↓
                          Exposed via: Conclusions API / Representations / Peer Cards / Chat Endpoint
```

### Internal Storage Detail
- Messages belong to sessions, labeled by source peer (user_peer_id or assistant_peer_id)
- Peer representations stored in **internal collections of vector-embedded documents**, keyed by `(observer, observed)` peer pairs
- Same mechanism powers both self-representation (`observer == observed`) and cross-peer modeling (peer X's understanding of peer Y)
- Collections are NOT directly API-accessible — the Conclusions API is the public surface
- Embeddings: uses OpenAI (when `EMBED_MESSAGES=true`) or pgvector native similarity

### Background Reasoning (Deriver)
The deriver worker processes:
1. `representation`: Updates per-peer representations (deductive + inductive conclusions)
2. `summary`: Session summarization for context compression
3. `dream`: Specialist models and surprisal-based processing for deeper insights
4. Observation settings: who observes whom (directional vs unified modes)

LLM providers used by the deriver (from `.env.template`):
- **Gemini**: deriver, summary, dialectic minimal/low (default)
- **Anthropic**: dialectic medium/high/max and dream processing
- **OpenAI**: embeddings (when enabled)

### Query Interface
Five query surfaces:

| Surface | Endpoint | Description |
|---------|----------|-------------|
| `peer.chat(query)` | Chat Endpoint | NL → synthesized answer via dialectic reasoning |
| `peer.representation()` | Representation | Static, low-latency snapshot of peer model |
| `peer.search(query)` | Hybrid Search | BM25 + vector search over stored messages |
| `session.context(tokens)` | Context | Summary + messages + peer representation bundle |
| Conclusions API | REST | Raw extracted facts (deductive + inductive) |

**Dialectic reasoning levels**: minimal → low → medium → high → max (controls LLM cost/depth tradeoff)

---

## 2. Self-Hosting: Exact Setup

### Requirements
- **Postgres with pgvector** (required)
- **Redis** (for queue/cache, per self-hosting docs and CHANGELOG)
- **LLM API keys** (required for reasoning):
  - `LLM_GEMINI_API_KEY` — deriver, summary, dialectic minimal/low
  - `LLM_ANTHROPIC_API_KEY` — dialectic medium/high/max, dream
  - `LLM_OPENAI_API_KEY` — embeddings (optional, only if `EMBED_MESSAGES=true`)
- Python 3.10+, uv 0.5.0+

### Docker Quick Start (Recommended)
```bash
git clone https://github.com/plastic-labs/honcho.git
cd honcho
cp docker-compose.yml.example docker-compose.yml
cp .env.template .env
# Edit .env — fill in at minimum:
#   DB_CONNECTION_URI=postgresql+psycopg://localhost/honcho_dev
#   LLM_GEMINI_API_KEY=...
#   LLM_ANTHROPIC_API_KEY=...
docker compose up
```
Docker Compose starts: Postgres (with pgvector), Redis, API server, and deriver worker.  
Note: **No pre-built Docker Hub image** — compose builds from source. Deriver startup is gated on API service healthcheck.

### Manual Development Setup (No Docker)
```bash
git clone https://github.com/plastic-labs/honcho.git
cd honcho
uv sync
source .venv/bin/activate

# 1. Database (use Docker for just Postgres)
cp docker-compose.yml.example docker-compose.yml
docker compose up -d database

# 2. Configure .env
cp .env.template .env
# Required:
export DB_CONNECTION_URI="postgresql+psycopg://localhost/honcho_dev"
export LLM_GEMINI_API_KEY="..."
export LLM_ANTHROPIC_API_KEY="..."
# Optional:
export AUTH_USE_AUTH=false   # disable auth for local dev
export SENTRY_ENABLED=false

# 3. Run migrations
uv run alembic upgrade head

# 4. Start API server (dev mode with auto-reload)
uv run fastapi dev src/main.py

# 5. Start deriver worker (separate terminal)
uv run python -m src.deriver
```

### Full Environment Variable Reference
```env
# REQUIRED
DB_CONNECTION_URI=postgresql+psycopg://localhost/honcho_dev

# LLM providers (at least Gemini required for basic reasoning)
LLM_GEMINI_API_KEY=
LLM_ANTHROPIC_API_KEY=
LLM_OPENAI_API_KEY=       # only if EMBED_MESSAGES=true

# Auth (disable for local dev)
AUTH_USE_AUTH=false
AUTH_JWT_SECRET=           # required if AUTH_USE_AUTH=true
                           # generate: python scripts/generate_jwt_secret.py

# Cache
# Redis config (section: [cache] in config.toml)

# Feature flags
EMBED_MESSAGES=false       # enable vector embeddings
SENTRY_ENABLED=false       # error tracking
METRICS_ENABLED=false      # Prometheus pull-based metrics
TELEMETRY_ENABLED=false    # CloudEvents telemetry

# Override any config.toml section via env:
# {SECTION}_{KEY} pattern
# e.g. DERIVER_MODEL_CONFIG__TRANSPORT, DIALECTIC_LEVELS__low__MODEL_CONFIG__MODEL
```

### Configuration Priority
`env vars > .env file > config.toml > defaults`

### config.toml Sections
`[app]`, `[db]`, `[auth]`, `[cache]`, `[llm]`, `[deriver]`, `[peer_card]`, `[dialectic]`, `[summary]`, `[dream]`, `[webhook]`, `[metrics]`, `[telemetry]`, `[vector_store]`, `[sentry]`

### Vector Store Options
- Default: **pgvector** (bundled with Postgres)
- Alternatives (configurable): turbopuffer, lancedb

### Deploy to Fly.io
Official docs at: `https://honcho.dev/docs/v3/contributing/self-hosting#deploying-on-fly-io`

### Resource Estimate
- Minimum: 1 Postgres instance with pgvector extension + Redis + 2 processes (API + deriver)
- Production: scale deriver horizontally for throughput
- For Hermes LXD container: ~512MB RAM for Postgres, ~128MB Redis, ~256MB per process ≈ ~1.5GB total minimum

---

## 3. Hermes Integration: Exact Mechanics

### Plugin Location
`/home/hermes/.local/share/pipx/venvs/hermes-agent/lib/python3.12/site-packages/plugins/memory/honcho/`

Files:
- `__init__.py` — `HonchoMemoryProvider` class (1327 lines)
- `cli.py` — Setup wizard, `cmd_setup`, `cmd_status`, `cmd_sessions`, etc. (1650 lines)
- `client.py` — `HonchoClientConfig` dataclass + config resolution (840 lines)
- `session.py` — `HonchoSessionManager` + `HonchoSession` (1341 lines)

### Setup Command
```bash
hermes memory setup honcho
# OR equivalently:
hermes honcho setup
```
The wizard:
1. Checks/installs `honcho-ai>=2.0.1`
2. Asks: cloud (`api.honcho.dev`) or local (self-hosted)
3. For cloud: prompts for API key, writes to `$HERMES_HOME/honcho.json`
4. For local: prompts for base URL (default `http://localhost:8000`), no API key needed
5. Asks for user peer name (your identity in Honcho)
6. Configures recall mode, dialectic settings, session strategy
7. Writes config to `~/.hermes/honcho.json`

### Environment Variables (Hermes-side)
| Variable | Purpose |
|----------|---------|
| `HONCHO_API_KEY` | Cloud API key (from `app.honcho.dev`). Falls back to config file. |
| `HONCHO_BASE_URL` | Self-hosted server URL. If set to a valid http/https URL, `api_key` is set to `"local"` (no auth sent). |
| `HONCHO_ENVIRONMENT` | `"production"` (default) or staging |
| `HONCHO_TIMEOUT` | HTTP timeout seconds for SDK calls (default: 30s) |
| `HERMES_HONCHO_HOST` | Override the active Honcho host key (skip profile detection) |

**Config resolution order (per `client.py`):**
```
1. $HERMES_HOME/honcho.json  (profile-local)
2. ~/.hermes/honcho.json     (default profile shared host blocks)
3. ~/.honcho/config.json     (global cross-app interop)
4. HONCHO_API_KEY / HONCHO_BASE_URL env vars (fallback)
```

### Plugin Hook Implementation
The `HonchoMemoryProvider` implements the `MemoryProvider` ABC:

| Hook | When Called | What Happens |
|------|------------|-------------|
| `initialize(session_id)` | Agent startup | Creates Honcho client, resolves session key, creates session, starts prewarm threads |
| `system_prompt_block()` | System prompt assembly | Returns static mode header (context/tools/hybrid) — prompt-cache friendly |
| `prefetch(query)` | **Before each LLM call** | Returns base context (representation + peer card) + dialectic supplement, assembled from background threads |
| `queue_prefetch(query)` | **After each turn** | Fires background threads to warm context and dialectic for next turn |
| `sync_turn(user, asst)` | **After each turn** | Async-queues user+assistant message to Honcho session |
| `on_turn_start(turn, msg)` | Turn start | Increments turn counter for cadence logic |
| `on_memory_write(action, target, content)` | Built-in memory tool writes | Mirrors `add user` writes as Honcho conclusions |
| `on_session_end(messages)` | Session exit / /reset | Flushes all pending messages to Honcho |
| `get_tool_schemas()` | Tool registration | Returns 5 tool schemas (unless context-only mode) |
| `handle_tool_call(name, args)` | Model tool call | Dispatches to honcho_profile/search/reasoning/context/conclude |
| `shutdown()` | Agent exit | Joins threads, flushes queue |

### The 5 Honcho Tools (registered with the LLM)
1. **`honcho_profile`** — Read/write peer card (curated fact list)
2. **`honcho_search`** — Hybrid BM25+vector search over stored context (raw, no synthesis)
3. **`honcho_reasoning`** — NL question → dialectic answer (levels: minimal/low/medium/high/max)
4. **`honcho_context`** — Full session context snapshot (summary + representation + card + recent msgs)
5. **`honcho_conclude`** — Write or delete a persistent conclusion about a peer

### What Data Gets Sent to Honcho
- **User messages** (sanitized with `sanitize_context()`) — sent after each turn
- **Assistant messages** (sanitized) — sent after each turn
- **MEMORY.md / USER.md / SOUL.md** — one-time migration on first session creation (skipped for `per-session` strategy)
- **Conclusions** — mirrored from built-in memory tool `add user` writes
- **Peer card updates** — when `honcho_profile` tool is called with `card` param

### Recall Modes
- **`hybrid`** (default): auto-injected context + tools available
- **`context`**: auto-injected only, tools hidden (cheaper, less flexible)
- **`tools`**: tools only, no auto-injection (agent pulls on demand), lazy session init

### Session Strategy Options
- **`per-directory`** (default): One Honcho session per working directory
- **`per-session`**: Fresh Honcho session per Hermes run (clean slate each time)
- Custom: configurable session mapping

### Observation Modes
- **`directional`** (default): Both user and AI observe each other (each builds separate model of the other)
- **`unified`**: AI observes user; user doesn't observe AI (simpler, less data)

---

## 4. Data Model: Workspaces, Peers, Sessions, Messages, Conclusions

```
Workspaces
├── Peers ←──────────────────────────────────────┐
│   ├── Sessions (many-to-many with peers)        │
│   └── Internal Collections (observer/observed)  │
│       └── Documents (vector-embedded)            │
│                                                  │
└── Sessions ←────────────────────────────────────┤
    ├── Peers (many-to-many)                       │
    └── Messages (labeled by source peer) ─────────┘
```

### Primitives

**Workspace** (formerly App):
- Top-level isolation boundary (like a namespace)
- Enables multi-tenant and multi-use-case separation
- Hermes default: workspace ID = "hermes" (or active profile name)

**Peer** (formerly User):
- ANY participant — human user OR AI agent
- ID pattern: `^[a-zA-Z0-9_-]+`
- Hermes maintains two peers per session: `user_peer_id` + `assistant_peer_id` (e.g. "hermes" or profile name)
- Per-peer observation config: `observe_me`, `observe_others` (per-session, via `SessionPeerConfig`)

**Session**:
- Conversation thread, many-to-many with peers
- Has per-peer observation configuration
- Hermes default key: working directory path (sanitized for Honcho pattern)
- Contains: messages + session-level context/summary

**Message**:
- Atomic data unit, always labeled by source peer
- Max 25,000 chars per message (configurable for self-hosted)
- Long messages chunked with `[continued]` prefix
- Stored with `created_at` timestamp for chronological ordering

**Conclusions**:
- Persistent extracted facts about a peer
- Both deductive (explicit) and inductive (reasoned)
- Stored in internal `(observer, observed)` collections
- Public surface: Conclusions API + peer.chat() + representations
- Can be explicitly created (`honcho_conclude`) or written by deriver background processing
- PII deletion supported via `delete_id` parameter

**Representations** (derived):
- Static, low-latency snapshots of what Honcho knows about a peer
- Updated asynchronously by deriver after new messages
- Scope: global (across all sessions) or session-scoped

**Peer Cards** (derived):
- Compact identity summaries (curated list of fact strings)
- Accumulate over time via dialectic cycles
- Not available on Honcho server < 3.x

**Session Context** (derived):
- Prompt-ready bundle: summary + representation + peer card + messages
- Respects token budget (`context_tokens`)
- Used by `honcho_context` tool and `prefetch()` base layer

---

## 5. Benchmarks: Pareto Frontier Claims and Actual Numbers

### Official Numbers (from honcho.dev/evals/ and plasticlabs.ai blog, Dec 19, 2025)

| Benchmark | Honcho Score | Notes |
|-----------|-------------|-------|
| **LongMem S** | **90.4%** | 92.6% with Gemini 3 Pro |
| **LoCoMo** | **89.9%** | Up from previous 86.9% |
| **BEAM** | Top scores across all tests | Exact % not in public search snippets |

### Competitor Landscape (from third-party sources, 2025-2026)
| System | LongMemEval | LoCoMo | Notes |
|--------|-------------|--------|-------|
| **Honcho** | ~90.4% | ~89.9% | State-of-the-art claim, Dec 2025 |
| **Mem0 v3** | ~94.8% | ~91.6% | Self-reported, Apr 2026 — disputed |
| **Hindsight (Vectorize)** | ~71.2%? | — | Vectorize-backed |
| **Zep** | ~63.8% | — | vs Mem0's 49% on GPT-4o |
| **MemPalace** | Claims 100% | Claims 100% | Apr 2026, disputed methodology |

### Pareto Frontier Claim
Honcho claims to dominate the Pareto frontier of **accuracy × cost × speed × token efficiency** — not just raw accuracy. The argument is: Honcho achieves SOTA accuracy *and* is the fastest, cheapest, most token-efficient option. Mem0 v3 may score higher on raw benchmarks but at higher cost and latency.

**Caveat**: Memory benchmark leaderboards are actively contested. Mem0 v3 (Apr 2026) may have surpassed Honcho on raw accuracy numbers. The "Pareto frontier" framing is from Dec 2025. Benchmark methodology and self-reporting are disputed (MemPalace claiming 100% perfect score is likely methodologically suspect).

### Benchmark Suite Context
- **LongMem / LongMemEval**: Long conversation memory recall over many turns
- **LoCoMo**: Longer Conversation Modeling — multi-session modeling of personality/facts
- **BEAM**: Honcho-specific evaluation framework for background reasoning quality

---

## 6. Privacy and Data Considerations

### What Data Leaves the Agent

**Cloud (api.honcho.dev)**:
- All conversation messages (user + assistant, sanitized)
- MEMORY.md, USER.md, SOUL.md contents (on first migration)
- Peer cards and conclusions
- Session metadata
- Sends to Honcho's cloud servers (Plastic Labs infrastructure)
- LLM reasoning happens via Anthropic/Gemini/OpenAI APIs (your data goes to these providers too, via Plastic Labs)

**Self-Hosted**:
- All data stays within your infrastructure
- LLM calls still go to external APIs (Gemini/Anthropic/OpenAI) UNLESS you configure local LLM endpoints
- No telemetry sent to Plastic Labs when `TELEMETRY_ENABLED=false`
- Redis + Postgres data is fully local
- **Full privacy control** if you also use local LLM providers for the deriver

### License Implications (AGPL-3.0)
- Honcho server is AGPL-3.0: if you self-host as part of a **networked service**, you must release your source code
- For personal use / internal deployment: AGPL imposes no practical restriction
- For a Hermes agent serving multiple users: technically requires open-sourcing your Hermes modifications if you distribute it as a service
- Client SDKs (`honcho-ai` Python package): license not AGPL — check separately

### Privacy Architecture Summary
| Deployment | Data residency | LLM calls | Telemetry |
|-----------|---------------|-----------|-----------|
| Cloud | Plastic Labs servers | Anthropic/Gemini/OpenAI via Plastic Labs | Honcho may collect |
| Self-hosted (cloud LLMs) | Your infra | Anthropic/Gemini/OpenAI directly | Off by default |
| Self-hosted (local LLMs) | Fully local | None (air-gapped) | Off |

### `sanitize_context()` 
Messages are passed through `sanitize_context()` before being sent to Honcho — this strips tool traces, internal metadata, and formats content for storage. The actual conversation content (user messages + assistant responses) is sent.

---

## 7. Limitations and Known Gaps

### Cold Start / Async Lag
- Background reasoning is **asynchronous**: newly stored messages take time to be reflected in representations
- First-turn dialectic may miss or time out (default 8s timeout for first-turn query)
- Empty representation on cold start (new session, no prior history)
- The plugin has elaborate warm/cold detection and prewarm threads to mitigate, but still noticeable on first run

### Peer Card Availability
- Peer cards only available on Honcho server **≥ 3.x**
- Self-hosted instances on older versions: `honcho_profile` returns empty with hint explaining why
- Cards accumulate over dialectic cycles — not immediate

### Benchmark Uncertainty
- Mem0 v3 (Apr 2026) claims to beat Honcho on LongMemEval/LoCoMo
- Honcho's Dec 2025 Pareto frontier claim may be outdated
- BEAM is Honcho's own benchmark — self-designed benchmarks favor the designer

### Configuration Complexity
- Many interacting settings: `recallMode`, `writeFrequency`, `sessionStrategy`, `dialecticDepth`, `dialecticCadence`, `contextCadence`, `observation`, `pinPeerName`, `userPeerAliases`, etc.
- Setup wizard helps but advanced tuning requires understanding the full config chain
- Hermes has legacy `base_url` vs `baseUrl` inconsistency (issue #2613)

### LLM Provider Dependencies
- Requires Gemini OR Anthropic API keys even for self-hosted (unless you configure local LLM endpoints)
- No official local LLM support in the deriver out of the box
- Uses specific models per reasoning level: changing providers requires config chain understanding

### Multi-User / Gateway
- Per-user peer scoping requires correct `pinPeerName` / `userPeerAliases` / `runtimePeerPrefix` config
- Without proper config, all users may share a single peer (memory leakage between users)

### Write Modes and Message Limits
- Default `write_frequency="async"` means messages write in background — potential loss on crash
- Message max: 25,000 chars (Honcho cloud); configurable for self-hosted
- Dialectic input max: 10,000 chars (configurable)

### Honcho Plugin Cron Guard
- Honcho is automatically disabled for cron/flush contexts (`agent_context in {"cron", "flush"}`)
- Prevents cron system prompts from corrupting user representations

### AGPL-3.0 License Risk
- If Hermes ever becomes a multi-user service, AGPL copyleft may require source disclosure

---

## 8. Concrete End-to-End Loop: Store → Reason → Query → Inject

### Hermes Runtime Flow (per turn)

```
STARTUP (once):
  initialize()
    → HonchoClientConfig.from_global_config()
    → get_honcho_client(cfg) → Honcho(workspace_id=..., api_key=..., base_url=...)
    → HonchoSessionManager.__init__()
    → get_or_create(session_key)  → honcho.session(id) + honcho.peer(user_id) + honcho.peer(ai_id)
    → session.add_peers([user, ai], [user_config, ai_config])
    → session.context(summary=True, tokens=N)  ← loads existing messages
    → prefetch_context(session_key)  ← background prewarm of representation + card
    → [thread] peer.chat("Who is this user?", reasoning_level="low")  ← dialectic prewarm

BEFORE TURN N (user sends message):
  on_turn_start(N, message)  → increment _turn_count
  prefetch(query)             → returns assembled context:
    Layer 1 (base context):
      session.representation() → static user representation text
      peer.peer_card()         → curated fact list
    Layer 2 (dialectic supplement):
      peer.chat(query, reasoning_level="low")  ← NL synthesis (from prewarm thread)
    → injected into system prompt as:
      "## Session Summary\n..." + "## User Representation\n..." + "## User Peer Card\n..." + dialectic

  [LLM CALL happens here with injected context]

AFTER TURN N (response generated):
  sync_turn(user_content, assistant_content)
    → session.add_message("user", cleaned_content)
    → session.add_message("assistant", cleaned_content)
    → async_queue.put(session)  ← background writer thread
        → honcho_session.add_messages([user_peer.message(...), ai_peer.message(...)])
        → Honcho stores messages → enqueues deriver task

  queue_prefetch(next_query)
    → [thread] prefetch_context(session_key)  ← refresh base layer
    → [thread] peer.chat(next_query, ...)      ← warm dialectic for next turn

BACKGROUND (Honcho server):
  Deriver worker:
    → Reads enqueued session
    → Runs LLM reasoning over messages (Gemini/Anthropic)
    → Updates peer representation (stored in internal collections)
    → Updates session summary
    → Updates peer card
    → Results available via Chat/Representation/Conclusions endpoints on next query

ON TOOL CALL (model uses honcho_reasoning):
  handle_tool_call("honcho_reasoning", {"query": "What languages does the user prefer?", "reasoning_level": "medium"})
    → manager.dialectic_query(session_key, query, reasoning_level="medium", peer="user")
    → peer.chat(query, reasoning_level="medium")
    → Returns synthesized NL answer

SESSION END:
  on_session_end(messages)
    → flush_all()  ← ensure all messages written to Honcho
```

### Python SDK Example (standalone)
```python
from honcho import Honcho

# Cloud
honcho = Honcho(workspace_id="hermes", api_key="hch-...")
# OR self-hosted:
honcho = Honcho(workspace_id="hermes", base_url="http://localhost:8000")

# 1. STORE
user = honcho.peer("alice")
ai = honcho.peer("hermes")
session = honcho.session("workspace-project-123")
session.add_peers([(user, user_config), (ai, ai_config)])

session.add_messages([
    user.message("I prefer Python over Go for scripting"),
    ai.message("Noted! I'll stick to Python for our automation work."),
])

# 2. REASON (happens asynchronously in background — deriver processes queue)
import time; time.sleep(2)  # wait for async processing

# 3. QUERY
answer = user.chat("What programming languages does this user prefer?", reasoning_level="low")
context = session.context(summary=True, tokens=10_000)
card = user.peer_card()

# 4. INJECT
messages = context.to_anthropic(assistant=ai)
# → [{"role": "user", "content": "..."}, {"role": "assistant", "content": "..."}]
# Prepend memory context to system prompt and call your LLM
```

---

## 9. Verdict: Is Honcho the Right First Prototype for Hermes Memory?

### Strengths for Hermes
| Factor | Assessment |
|--------|-----------|
| **Already installed** | Plugin is live in hermes-agent venv; `hermes memory status` shows it available |
| **Native Hermes integration** | Best-in-class: wizard, profile cloning, cron guard, lazy init, 5 tools, all lifecycle hooks |
| **Self-host path clear** | Docker Compose + Postgres + Redis; AGPL but private use is fine |
| **Peer-centric model** | Perfect for agent memory — models both user and AI as peers |
| **Multi-recall modes** | hybrid/context/tools gives cost-flexibility |
| **Reasoning quality** | SOTA on LongMem + LoCoMo (Dec 2025); dialectic reasoning extracts real insights |
| **Single-user LXD container** | No multi-user peer leakage risk; `pinPeerName` covers cross-platform unification |
| **Migration from MEMORY.md** | One-time automatic migration on first new session |
| **Active development** | v3.0.9, actively maintained by Plastic Labs |

### Risks / Concerns
| Factor | Risk Level | Mitigation |
|--------|-----------|------------|
| **LLM API key dependency** | Medium | Need Gemini + Anthropic keys even for self-hosted; adds cost |
| **Cold start lag** | Low | Plugin pre-warms at init; prewarm thread runs before first turn |
| **Async reasoning gap** | Low | Static `representation` endpoint bridges gap; first-turn empty acceptable |
| **Benchmark claims contested** | Low | Use cases are personal agent memory, not benchmark races; quality good enough |
| **Setup complexity** | Low | `hermes honcho setup` wizard handles it; minimal manual config |
| **AGPL license** | Very Low | Personal/internal use; no redistribution |
| **No honcho.json currently** | None | First-time setup wizard will create it |

### Recommendation: **YES — Prototype Honcho Self-Host First**

**Rationale**:
1. The Hermes plugin is already installed and fully wired — zero new code required
2. `hermes honcho setup` will create `~/.hermes/honcho.json` with a 2-minute interactive flow
3. For cloud: just needs `HONCHO_API_KEY` → `echo 'HONCHO_API_KEY=hch-...' >> ~/.hermes/.env`
4. For self-host: `git clone` + fill `.env` (Gemini + Anthropic keys) + `docker compose up`
5. Addresses the core gap in Hermes memory: **cross-session semantic recall + user modeling**
6. Built-in memory (MEMORY.md/USER.md) remains active in parallel — Honcho is additive
7. The 3-layer system (built-in memory + Honcho context injection + Honcho tools) is the optimal Hermes memory stack

### Recommended First Step Sequence
```bash
# Option A: Cloud (fastest to test)
echo 'HONCHO_API_KEY=hch-...' >> ~/.hermes/.env  # get key at app.honcho.dev
hermes memory setup honcho   # or: hermes honcho setup
hermes memory status          # verify honcho shows as active provider

# Option B: Self-hosted LXD container (full privacy)
# 1. On the host or in a dedicated LXD container:
git clone https://github.com/plastic-labs/honcho.git
cd honcho && cp docker-compose.yml.example docker-compose.yml && cp .env.template .env
# edit .env: DB_CONNECTION_URI, LLM_GEMINI_API_KEY, LLM_ANTHROPIC_API_KEY
docker compose up -d

# 2. Back in Hermes:
hermes honcho setup
# → choose "local" → base URL: http://localhost:8000 (or container IP:8000)
hermes memory status

# Validate
# In a new session, chat for a few turns, then:
# "Use honcho_reasoning to tell me what you know about my preferences"
# "Use honcho_profile to show my peer card"
```

---

## Appendix: Current Hermes State

- **Honcho plugin**: Installed at `plugins/memory/honcho/` in hermes-agent venv
- **No active config**: No `~/.hermes/honcho.json` found, no `HONCHO_API_KEY` in `.env`
- **Status**: Plugin present but not configured (shows in `hermes memory status` as available)
- **Other providers**: byterover, hindsight, holographic, mem0, openviking, retaindb, supermemory also installed
- **LLM providers configured**: OpenAI-Codex (primary), Anthropic + OpenRouter (fallbacks)

The Hermes honcho plugin is production-ready with 3,817 lines of carefully engineered Python across 4 files covering the full lifecycle. It is the most deeply integrated memory provider in the Hermes ecosystem.
