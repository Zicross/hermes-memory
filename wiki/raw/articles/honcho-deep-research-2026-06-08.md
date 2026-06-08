---
source_url: internal-research
ingested: 2026-06-08
sha256: research-report
---

# Honcho Memory System — Deep Research Report

**Research date:** 2026-06-08  
**Purpose:** Evaluate Honcho as the first external semantic memory provider for Hermes Agent  
**Sources:** Honcho README (already ingested), Hermes plugin source code (live), Hermes docs (web), Plastic Labs benchmarks blog (web)

---

## Table of Contents

1. [Architecture](#architecture)
2. [Self-Hosting Setup](#self-hosting)
3. [Hermes Integration Mechanics](#hermes-integration)
4. [Data Model](#data-model)
5. [Benchmarks & Evals](#benchmarks)
6. [Privacy & Data Considerations](#privacy)
7. [Limitations & Known Gaps](#limitations)
8. [End-to-End Loop Example](#end-to-end-example)
9. [Verdict](#verdict)

---

## 1. Architecture {#architecture}

Honcho splits into **two cooperating services**:

### Storage Service (synchronous, via HTTP API)
- Stores workspaces, peers, sessions, messages, and internal document collections
- FastAPI server; all writes return immediately
- Backend: PostgreSQL + pgvector for vector storage

### Insights Service (asynchronous, via background worker — "deriver")
- Consumes a queue of messages posted by the API
- Runs background reasoning over conversation history
- Updates **peer representations**, **session summaries**, and **peer cards**
- Separate process: `uv run python -m src.deriver`

### Reasoning Pipeline (the "deriver")
1. Messages arrive via API → enqueued for processing
2. Deriver picks up tasks: `representation`, `summary`, dream tasks
3. Runs LLM reasoning (Gemini / Anthropic / OpenAI depending on task/level)
4. Stores results in **internal collections** keyed by `(observer, observed)` peer pairs
5. Results surfaced via: Conclusions API, Representations endpoint, Peer Cards, Chat Endpoint

### Query Interface

| Interface | Description | Latency |
|-----------|-------------|---------|
| `peer.chat(query)` | Dialectic reasoning — LLM call against all known observations | Medium (~2-5s cloud) |
| `peer.representation()` | Static snapshot of current peer representation | Low (<200ms) |
| `peer.search(query)` | Hybrid BM25 + vector search over stored observations | Low |
| `session.context(summary, tokens)` | Prompt-ready bundle: messages + summary + representation | Low |
| Conclusions API | Raw stored observations about a peer | Low |

### Background Reasoning Quality Tiers
The dialectic (chat endpoint) accepts a `reasoning_level` parameter:
- `minimal` — quick factual lookup (Gemini-class, cheap)
- `low` — default for Hermes integration
- `medium` — multi-aspect synthesis
- `high` — complex behavioral patterns
- `max` — exhaustive audit-level analysis (Anthropic, expensive)

---

## 2. Self-Hosting Setup {#self-hosting}

Honcho is **open source under AGPL-3.0**.

### Quick Start (Docker — recommended)

```bash
git clone https://github.com/plastic-labs/honcho.git
cd honcho
cp docker-compose.yml.example docker-compose.yml
cp .env.template .env     # edit with your LLM API keys
docker compose up
```

Server listens on **http://localhost:8000** by default.

### Required Services (from Docker Compose)

| Service | Role |
|---------|------|
| **PostgreSQL + pgvector** | Primary data store; messages, peers, sessions, vector embeddings |
| **Redis** (optional cache) | Caching layer (`[cache]` config section); not required for basic operation |
| **Honcho API server** | FastAPI HTTP server |
| **Deriver worker** | Background reasoning process (separate from API) |

> **Note:** Redis shows up in Docker Compose and in the `[cache]` config section, but per the README and source it's optional — the cache section defaults to disabled. The **critical** services are Postgres + pgvector and the deriver.

### Required Environment Variables

```env
# Database
DB_CONNECTION_URI=postgresql+psycopg://user:pass@localhost/honcho
# Note: MUST use `postgresql+psycopg` prefix (SQLAlchemy requirement)

# LLM API Keys (use at least one)
LLM_GEMINI_API_KEY=     # Used for: deriver, summary, dialectic minimal/low by default
LLM_ANTHROPIC_API_KEY=  # Used for: dialectic medium/high/max and dream
LLM_OPENAI_API_KEY=     # Used for: embeddings when EMBED_MESSAGES=true

# Auth (optional — disable for local dev)
AUTH_USE_AUTH=false
SENTRY_ENABLED=false
```

### Authentication for Self-Hosted
- `AUTH_USE_AUTH=false` → no auth required, no API key needed
- `AUTH_USE_AUTH=true` → requires JWT. Generate secret: `python scripts/generate_jwt_secret.py`, set `AUTH_JWT_SECRET=<secret>`

### Non-Docker Local Setup (for development)

Requirements: Python ≥3.10, uv ≥0.5.0

```bash
git clone https://github.com/plastic-labs/honcho.git
cd honcho
uv sync
cp docker-compose.yml.example docker-compose.yml
docker compose up -d database         # Just Postgres

cp .env.template .env                 # fill in keys
uv run alembic upgrade head           # run migrations
uv run fastapi dev src/main.py        # API server (dev, hot-reload)

# In second terminal:
uv run python -m src.deriver          # Background worker
```

### Configuration Priority Order
`ENV vars > .env file > config.toml > defaults`

### config.toml Sections Available
`[app]`, `[db]`, `[auth]`, `[cache]`, `[llm]`, `[deriver]`, `[peer_card]`, `[dialectic]`, `[summary]`, `[dream]`, `[webhook]`, `[metrics]`, `[telemetry]`, `[vector_store]`, `[sentry]`

### Resource Requirements
- **Minimum**: Postgres + pgvector, 1 LLM API key (Gemini works for basic use)
- **RAM**: No hard figure from docs; Postgres + FastAPI + deriver worker is reasonable at ~512MB-1GB on a small server
- **Disk**: Grows with conversation history; pgvector indexes scale with message volume
- **Vector store backends**: pgvector (default), turbopuffer, lancedb

---

## 3. Hermes Integration Mechanics {#hermes-integration}

### Plugin Location
`/home/hermes/.local/share/pipx/venvs/hermes-agent/lib/python3.12/site-packages/plugins/memory/honcho/`

Files:
- `__init__.py` — `HonchoMemoryProvider(MemoryProvider)` — main integration
- `cli.py` — setup wizard, profile management CLI
- `client.py` — `HonchoClientConfig`, config resolution, SDK initialization
- `session.py` — `HonchoSessionManager`, `HonchoSession`, async write queue

### Setup Commands

```bash
# Generic memory setup wizard (selects provider from list)
hermes memory setup
# → Select "honcho" → runs same wizard

# Honcho-specific wizard (direct)
hermes honcho setup

# Check status
hermes memory status
hermes honcho status

# Multi-profile sync
hermes honcho sync
```

### Config File Resolution Order
1. `$HERMES_HOME/honcho.json` (profile-local, e.g. `~/.hermes/honcho.json`)
2. `~/.honcho/config.json` (global, cross-app interop)
3. Environment variables (`HONCHO_API_KEY`, `HONCHO_BASE_URL`, etc.)

### Key Environment Variables

| Variable | Purpose |
|----------|---------|
| `HONCHO_API_KEY` | API key for Honcho cloud (`api.honcho.dev`). Required for cloud, optional for no-auth self-hosted |
| `HONCHO_BASE_URL` | Override base URL; point at self-hosted: `http://localhost:8000` |
| `HONCHO_ENVIRONMENT` | `production` (default) or `local` |
| `HONCHO_TIMEOUT` | HTTP request timeout in seconds (default: 30.0) |
| `HERMES_HONCHO_HOST` | Override active host key (profile routing) |

**For self-hosted no-auth:** Set `HONCHO_BASE_URL=http://localhost:8000`. No API key needed — the plugin uses `"local"` as a sentinel when `base_url` is present.

### Plugin Lifecycle Hooks (called by Hermes MemoryManager)

| Hook | When Called | What Honcho Does |
|------|------------|-----------------|
| `initialize(session_id, **kwargs)` | Agent startup | Resolves config, creates SDK client, creates/resumes Honcho session, migrates MEMORY.md/USER.md/SOUL.md on first session, starts prewarm thread |
| `system_prompt_block()` | System prompt assembly | Returns static mode header (hybrid/context/tools instructions) — prompt-cache friendly |
| `prefetch(query)` | Before each LLM call | Returns assembled context: base layer (representation + card) + dialectic supplement |
| `queue_prefetch(query)` | After each turn | Fires background threads for next turn's context refresh and dialectic call |
| `on_turn_start(turn_n, message)` | Turn start | Updates turn counter for cadence logic |
| `sync_turn(user_content, asst_content)` | After each turn | Writes conversation turn to Honcho (async background thread, with retry) |
| `on_memory_write(action, target, content)` | On built-in memory write | Mirrors `add/user` writes as Honcho conclusions |
| `on_session_end(messages)` | Session end | Flushes all pending messages |
| `get_tool_schemas()` | Tool registration | Returns 5 tool schemas (or 0 in context-only mode) |
| `handle_tool_call(tool_name, args)` | Tool call | Dispatches to Honcho SDK |
| `shutdown()` | Agent exit | Joins threads, flushes all |

### Tools Exposed to the Model (in hybrid or tools mode)

| Tool | Description | Cost |
|------|-------------|------|
| `honcho_profile` | Read or write peer card (key facts: name, role, prefs) | Free (static) |
| `honcho_search` | Hybrid BM25+vector search over stored observations | Low |
| `honcho_context` | Full session snapshot: summary + representation + card + recent messages | Low |
| `honcho_reasoning` | Natural-language dialectic query with configurable depth | Medium-High (LLM call) |
| `honcho_conclude` | Write or delete a conclusion (persistent fact) | Low |

### Recall Modes
- **`hybrid`** (default) — auto-injected context AND tools available
- **`context`** — context injected automatically, no tools
- **`tools`** — tools only, no auto-injection (lazy session init)

### Data Written Per Turn
1. **Messages**: sanitized user + assistant content, chunked at 25k chars if needed
2. **Conclusions**: mirrored from built-in memory writes
3. **Session context**: maintained via Honcho session API

### Session Strategy Options
- `per-directory` (default) — one session per working directory
- `per-session` — fresh Honcho session per Hermes run

---

## 4. Data Model {#data-model}

```
Workspace (top-level, isolates use cases)
├── Peers (humans OR AI agents — unified model)
│   ├── Internal collections (keyed by observer/observed pair)
│   │   └── Vector-embedded Documents → exposed as Conclusions
│   └── Peer Card (compact identity summary)
└── Sessions (conversations, many-to-many with Peers)
    ├── Messages (atomic, labelled by source peer)
    └── Summary (generated by deriver)
```

### Entity Descriptions

**Workspace** — top-level multi-tenant container. Each Hermes profile gets its own workspace (default: `"hermes"`). Isolates data between use cases.

**Peer** — any participant: human user, AI agent. First-class entity. Has `(observer, observed)` internal collections for cross-peer modeling (e.g., what the AI peer knows about the user peer vs. what the user knows about the AI).

**Session** — conversation context. Many-to-many with peers. Has observation configuration per peer pair (see `SessionPeerConfig`).

**Message** — atomic data unit. Lives on a session. Labelled by source peer. Enqueued for background reasoning on creation.

**Conclusions** — public surface for what Honcho has extracted about a peer. Stored internally in vector-embedded document collections, exposed via Conclusions API. Two types: deductive (explicit facts) and inductive (patterns reasoned from behavior).

**Representations** — static, low-latency snapshots of the peer model. Generated by deriver from conclusions. Used for fast injection without LLM call.

**Peer Cards** — compact identity summaries. Manually curated or auto-extracted. Compact subset of representation.

**Session Summary** — deriver-generated summary of what happened in a session.

---

## 5. Benchmarks & Evals {#benchmarks}

Source: https://honcho.dev/evals/ + https://blog.plasticlabs.ai/research/Benchmarking-Honcho (Dec 2025)

### Honcho v3 Scores

| Benchmark | Honcho Score | Notes |
|-----------|-------------|-------|
| **LongMem S** | **90.4%** | 92.6% with Gemini 3 Pro |
| **LoCoMo** | **89.9%** | Previous score: 86.9% |
| **BEAM** | Top scores across all tests | Specific % not surfaced in search results |

### Context from Competitive Landscape

Based on web research (April-June 2026 sources):

| System | LongMemEval | LoCoMo | Notes |
|--------|------------|--------|-------|
| Honcho v3 | ~90.4% | ~89.9% | Pareto frontier claim (Dec 2025) |
| Mem0 v3 | ~94.8% | ~91.6% | Claimed Apr 2026 (self-reported) |
| Hindsight (Vectorize) | Comparable to Mem0 v2 | - | RAG-based |
| Zep | 63.8% | - | GPT-4o |
| MemPalace | "500/500" LongMemEval | 100% LoCoMo | Apr 2026 — claimed perfect; methodology disputed |

> **Caveat:** The "Pareto frontier" claim from Dec 2025 predates Mem0 v3 and MemPalace's April 2026 claims. The benchmark landscape is evolving rapidly. Honcho's key differentiator is the **combination** of accuracy + speed + cost + token efficiency, not raw accuracy alone. Self-reported benchmarks from all vendors should be treated with skepticism.

### Honcho's Pareto Claim
Plastic Labs claims Honcho is simultaneously:
- Highest accuracy across LongMem, LoCoMo, BEAM
- Fastest memory solution
- Cheapest / most token-efficient
- State-of-the-art at time of Dec 2025 release

---

## 6. Privacy & Data Considerations {#privacy}

### Cloud (`api.honcho.dev`)
- All conversation content, user messages, assistant responses sent to Honcho servers
- Honcho uses LLM APIs (Gemini, Anthropic, OpenAI) to process data — conversations pass through these providers too
- Plastic Labs's privacy policy governs data retention and use
- Data residency: Honcho cloud (US-based)
- **Not appropriate for sensitive/confidential conversations**

### Self-Hosted
- **Nothing leaves your machine** — all data stays local
- You control which LLM providers the deriver uses (can point at local models)
- You own the Postgres database
- AGPL-3.0 license: if you deploy self-hosted Honcho as part of a networked service/application, you must open-source your application code (strong copyleft)
- For personal CLI use (Hermes), AGPL is generally fine — it's internal, not a distributed service

### What Data Is Sent (cloud or self-hosted)
Per turn, Honcho receives:
1. User message content (sanitized via `sanitize_context()` — removes some PII patterns)
2. Assistant response content
3. Session/peer identifiers (not personally identifying by default)
4. Occasionally: MEMORY.md, USER.md, SOUL.md content on first session (migration feature)

### Self-Hosted vs Cloud Key Differences
| Aspect | Cloud | Self-Hosted |
|--------|-------|-------------|
| Data residency | Honcho servers | Your machine |
| Setup | 1 API key | Docker + Postgres + LLM keys |
| Auth required | Yes (API key) | Optional (AUTH_USE_AUTH=false) |
| LLM costs | Included in Honcho pricing | You pay your LLM providers directly |
| AGPL compliance | N/A | Applies if distributing |
| Reliability | Managed uptime | Your responsibility |

---

## 7. Limitations & Known Gaps {#limitations}

### Async Reasoning Latency
- **Background reasoning is always async** — newly added messages may take seconds to minutes before they're reflected in `peer.chat()` / representation queries
- First-turn context is often empty or stale for new users with little history
- Plugin compensates with a cold-start prewarm query at session init, but this still adds ~2-8s of background latency before first meaningful dialectic result

### Representation Freshness
- `peer.representation()` is fast but static — it's a snapshot from the last deriver run
- Busy sessions may have a lag between what was said and what Honcho "knows"

### Cold Start / Sparse Data Problem
- Peer cards and representations are empty for new users
- Honcho's dialectic reasoning requires enough conversation history to be useful
- The `empty_profile_hint` in the plugin gracefully handles this, but the model still gets "No profile facts available yet" for new users

### Self-Hosting Complexity
- Requires Postgres with pgvector (not just plain Postgres)
- Requires running two services: API server + deriver worker
- No pre-built Docker Hub image — must build from source
- LLM API keys required (at minimum Gemini for basic operation)

### AGPL License
- AGPL-3.0 means forking and deploying publicly requires open-sourcing your modifications
- Fine for personal/internal Hermes use; problematic if wrapping in a commercial product

### Benchmark Landscape
- Honcho's Dec 2025 "Pareto frontier" claim is being challenged by newer systems (Mem0 v3, MemPalace) as of April-June 2026
- All vendors run their own benchmarks — independent evaluations are scarce

### No Offline Mode
- Even self-hosted Honcho needs external LLM API calls for reasoning
- Pointing at local LLMs (Ollama) requires custom configuration

### Reasoning Cost
- `dialectic` (chat endpoint) at `medium`+ levels uses Anthropic Claude — adds latency and cost per query
- Default is `low` (Gemini), which is cheaper but less deep

---

## 8. End-to-End Loop Example {#end-to-end-example}

Here's the full Honcho loop as it operates in Hermes:

### 1. STORE (Session Init)

```python
# On Hermes startup, plugin calls initialize()
# Session key resolved (e.g. per-directory strategy)
session_key = "cli:/home/hermes"

# Honcho client created with workspace="hermes", peer="user-peer-name"
honcho = Honcho(workspace_id="hermes", base_url="http://localhost:8000")
user_peer = honcho.peer("alice")       # get_or_create
ai_peer   = honcho.peer("hermes")      # get_or_create
session   = honcho.session("cli--home-hermes")  # get_or_create

# First session: MEMORY.md / USER.md content migrated as messages
# Observation configured: who observes whom
session.add_peers([(user_peer, observe_me=True), (ai_peer, observe_me=True)])
```

### 2. REASON (Background Deriver)

```
After messages arrive → deriver picks up queue item:
  Task: representation update for peer "alice"
  → LLM analyzes all messages (Gemini for low/minimal, Anthropic for high/max)
  → Extracts observations: "Alice works in Python, prefers concise answers,
     is working on an LXD homelab setup"
  → Stores as vector-embedded conclusions in collection(observer=hermes, observed=alice)
  → Updates alice.representation (static snapshot)
  → Updates alice.peer_card
```

### 3. QUERY (Turn Start Prefetch)

```python
# Before each LLM call, plugin calls prefetch(user_query)
# Layer 1: base context (sync on first call, cached after)
ctx = session.context(summary=True, tokens=10_000)
representation = ctx.representation   # "Alice is a Python developer..."
card = user_peer.peer_card()          # Compact facts

# Layer 2: dialectic supplement (background thread)
# query = "help me set up my LXD container networking"
dialectic = user_peer.chat(
    "Given what's been discussed, what context about this user is most relevant?",
    reasoning_level="low"
)
# → Synthesized: "Alice is setting up an LXD homelab. Recent work has been 
#    focused on container networking. She prefers direct answers with code examples."
```

### 4. INJECT (Into LLM Call)

```python
# Assembled context injected into system prompt:
system = f"""
## User Representation
{representation}

## User Peer Card
{card}

{dialectic}
"""

# Main LLM call with enriched context
response = client.chat(
    model="gpt-5.5",
    system=system,
    messages=[{"role": "user", "content": user_query}]
)
```

### Post-Turn: Write Back

```python
# After each turn (async background thread)
session.add_messages([
    user_peer.message(user_query),
    ai_peer.message(response),
])
# → Enqueues for next deriver reasoning cycle
```

---

## 9. Verdict {#verdict}

### Should Honcho be the first prototype for Hermes external memory?

**Yes — with the self-hosted path.**

#### Reasons For:

1. **Already integrated**: Plugin code is already installed and production-quality. `hermes honcho setup` runs a full wizard. No custom code needed.

2. **Architecture fit**: Honcho's peer-centric model maps directly to Hermes's use case — one user (the operator), one AI peer (Hermes), persistent cross-session model of the user.

3. **Deepest reasoning**: Unlike simple vector RAG, Honcho extracts *conclusions* — what it knows about the user — via dialectic LLM reasoning. This aligns with "insanely good memory" goal.

4. **Privacy-first self-host option**: Self-hosted Honcho + no-auth = zero data leaving the machine. Local LXD container deployment is feasible.

5. **Tools for agent-driven memory**: The 5 honcho tools (`profile`, `search`, `context`, `reasoning`, `conclude`) let Hermes actively manage its memory, not just passively consume injection.

6. **Benchmark credibility**: ~90% on LongMem, ~90% on LoCoMo at time of Dec 2025 release is genuinely competitive (Mem0/Hindsight were lower). Newer claims (Apr 2026) suggest the gap narrowed, but Honcho was purpose-built for agent use.

7. **Hermes already has the code, the docs, and the planning notes** for this.

#### Reasons Against / Risks:

1. **Self-host complexity**: Postgres+pgvector + API server + deriver worker + LLM API keys is 4 moving parts. Non-trivial for a homelab.

2. **Async reasoning lag**: Fresh Hermes sessions will have empty/stale context until enough history accumulates. Could feel broken at first.

3. **LLM cost**: Honcho's reasoning calls hit LLM APIs. Low-level Gemini is cheap but not free. High-quality reasoning (Anthropic) adds up.

4. **AGPL license**: Fine for personal use, but a constraint for any future distribution.

5. **Benchmark competition**: Dec 2025 Pareto claim is contested by Apr 2026 Mem0 v3 numbers. But Hermes integration quality > raw benchmark score.

#### Recommended Config for Hermes Prototype

```bash
# Self-hosted, no-auth, Gemini for reasoning (cheapest)
docker compose up   # in plastic-labs/honcho clone

hermes honcho setup
# → local
# → Base URL: http://localhost:8000
# → Your name: [your name]
# → AI peer: hermes
# → Recall mode: hybrid
# → Dialectic level: low
```

```env
# In ~/.hermes/.env (or honcho.json)
HONCHO_BASE_URL=http://localhost:8000
LLM_GEMINI_API_KEY=your-gemini-key  # For deriver (in Honcho .env)
```

**Bottom line**: Honcho is the right first external memory prototype. It's the only provider with native deep reasoning (vs. RAG), it's already fully wired into Hermes, the self-host path is real and tested, and it's built specifically for the "stateful AI agent with persistent user understanding" use case that Hermes needs.
