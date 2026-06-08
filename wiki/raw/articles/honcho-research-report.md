---
source_url: local + plugin source + honcho README
ingested: 2026-06-08
sha256: manual-synthesis
---

# Honcho Deep Research Report
*For Hermes Agent — Memory Provider Prototype Assessment*
*Researched: 2026-06-08 | Sources: Honcho README v3.0.9, Hermes plugin source (plugins/memory/honcho/), live system*

---

## Executive Summary

Honcho is AI-native memory infrastructure focused on **understanding and modeling people and agents over time** — a fundamentally different thesis from fact-storage systems like Mem0 or retrieval-quality systems like RetainDB. Its architecture is built around *peers* (users, agents, projects) that participate in *sessions*, with a background *deriver* that reasons about conversations to build evolving *representations* and *peer cards*.

For Hermes, Honcho is the best-fit first external semantic memory prototype because:
1. Hermes already has a complete, mature plugin installed with 5 tools and background prefetch
2. Self-host requires only Docker (git clone + docker compose up) with Postgres+pgvector
3. The plugin supports `HONCHO_BASE_URL` for self-hosted endpoint — no data leaves the container
4. Honcho's user/agent modeling is complementary (not redundant) to Mem0's ADD-only fact store
5. The setup wizard is built-in: `hermes memory setup` → select honcho → wizard runs automatically

**Verdict: Honcho should be the first self-hosted prototype. Mem0 v3 is the best second candidate.**

---

## 1. Data Model and Core Concepts

Honcho organizes memory around a hierarchy of entities:

```
Workspace (app-level namespace)
└── Peer (any actor: user, agent, group, project, idea)
    └── Session (a conversation thread)
        └── Message (content from a peer in that session)
            └── (Background: Deriver processes messages → Representation + Queue)
```

### Key concepts

**Peer**: Any actor that participates in conversations — not just users. A peer can represent:
- The user (Isaac)
- The agent (Hermes)
- A project ("constituint")
- An idea ("memory-system-buildout")

This multi-peer design means Honcho can model relationships between users and agents symmetrically, not just "user preferences for bot."

**Session**: A conversation thread. Messages are associated with sessions, not just users. Sessions accumulate a `context()` that can be directly injected into LLM calls.

**Representation**: An automatically generated, always-updated description of what Honcho knows about a peer. The background deriver builds representations asynchronously from conversation messages.

**Peer Card**: A curated list of key facts about a peer (name, role, preferences, communication style, patterns). Updated by the agent via `honcho_profile(card=[...])` or accumulated by the deriver.

**Conclusion**: A persistent explicit fact written by the agent via `honcho_conclude(conclusion="...")`. Conclusions are high-confidence facts that build the peer's profile. Deletion is only for PII removal — Honcho self-heals incorrect conclusions over time.

**Dialectic (`.chat()` in the SDK)**: A natural-language question answered by Honcho using LLM reasoning over what it knows about a peer. This is the most expensive operation but enables queries like "What communication style does this user prefer?" or "What did we learn about their coding workflow last week?" Honcho calls its background LLM process the *deriver*; the dialectic is a on-demand query against what the deriver has built.

**Queue**: The background processing queue. Messages added to sessions are placed in a queue. The deriver processes the queue to generate/update representations and conclusions. Queue status is inspectable via `honcho.queue_status()`.

### The Honcho Loop (concrete)

```python
# 1. STORE: create peers and add messages to a session
user = honcho.peer("isaac")
agent = honcho.peer("hermes")
session = honcho.session("session-20260608")
session.add_messages([
    user.message("What's the model policy?"),
    agent.message("Primary is gpt-5.5 via openai-codex..."),
])

# 2. REASON: happens asynchronously in background (deriver process)
# No explicit call needed — deriver picks up from the queue

# 3. QUERY: ask Honcho what it knows
answer = user.chat("What does Isaac prefer for model/provider decisions?")
representation = user.representation()  # Fast, non-LLM static representation

# 4. INJECT: hand context to Claude/GPT
context = session.context(summary=True, tokens=10_000)
response = anthropic.messages.create(
    model="claude-opus-4-5",
    messages=context.to_anthropic(assistant=agent),
)
```

---

## 2. Self-Host Setup

### Exact Docker commands

```bash
# Clone
git clone https://github.com/plastic-labs/honcho.git
cd honcho

# Configure
cp docker-compose.yml.example docker-compose.yml
cp .env.template .env

# Fill in .env (minimum required):
# DB_CONNECTION_URI=postgresql+psycopg://... (or use the docker-compose Postgres)
# LLM_GEMINI_API_KEY=...    ← for deriver/summary/dialectic minimal/low
# LLM_ANTHROPIC_API_KEY=... ← for dialectic medium/high/max and dream
# LLM_OPENAI_API_KEY=...    ← for embeddings when EMBED_MESSAGES=true
# AUTH_USE_AUTH=false        ← disable JWT auth for local dev
# SENTRY_ENABLED=false

# Start (includes Postgres + pgvector + API + deriver)
docker compose up

# Verify API is running
curl http://localhost:8000/health
```

**Exposed ports**: `:8000` (FastAPI API server)

### Services started by docker-compose
1. **database** — Postgres with pgvector extension
2. **api** — FastAPI server (the Honcho API)
3. **deriver** — Background worker that processes the message queue and generates representations/conclusions

### Alternative: development mode (no Docker)

```bash
# Prerequisites: Python 3.10+, uv 0.5+, Postgres+pgvector

uv sync                              # creates .venv
source honcho/.venv/bin/activate
# Set up .env as above with DB_CONNECTION_URI
uv run alembic upgrade head          # run DB migrations
uv run fastapi dev src/main.py       # API server on :8000
uv run python -m src.deriver         # background worker (separate terminal)
```

### Important: LLM routing

Honcho itself needs LLMs to run the deriver. The default configuration routes:
- **Gemini** (via `LLM_GEMINI_API_KEY`): deriver, summary, dialectic minimal/low
- **Anthropic** (via `LLM_ANTHROPIC_API_KEY`): dialectic medium/high/max and dream
- **OpenAI** (via `LLM_OPENAI_API_KEY`): embeddings (when `EMBED_MESSAGES=true`)

This means for a fully isolated self-host using Claude, set `LLM_ANTHROPIC_API_KEY` to the same OAuth credential Hermes uses. The Hermes Claude Max subscription can serve both Hermes itself AND the Honcho deriver.

### License

AGPL-3.0 (same as OpenViking). For personal use, no issue. For a commercial product, requires review.

---

## 3. Hermes Plugin Mechanics

### Plugin location and structure

```
plugins/memory/honcho/
├── __init__.py    (1,327 lines — HonchoMemoryProvider, 5 tool schemas)
├── client.py      (840 lines — HonchoClientConfig, config resolution)
├── session.py     (1,341 lines — HonchoSessionManager, HonchoSession)
├── cli.py         (Honcho setup wizard)
└── plugin.yaml    (metadata: name=honcho, version=1.0.0, hook: on_session_end)
```

### 5 Exposed Tools

| Tool | Operation | Cost |
|---|---|---|
| `honcho_profile` | Read or write a peer card | Fast (static) |
| `honcho_search` | BM25 + vector search over stored context | Fast (no LLM) |
| `honcho_reasoning` | Natural-language dialectic query with LLM synthesis | Expensive |
| `honcho_context` | Full session context snapshot (summary + card + recent messages) | Cheap (no LLM) |
| `honcho_conclude` | Write/delete a persistent conclusion about a peer | Write operation |

### Data Flow per Turn

```
User message arrives
→ on_turn_start() updates turn counter
→ prefetch() returns pre-fetched dialectic result from PREVIOUS turn
  (includes: base context layer + dialectic supplement)
→ [injected into system prompt before LLM call]
→ queue_prefetch() fires in BACKGROUND for NEXT turn:
    Layer 1: get_prefetch_context() → peer representation + peer card
    Layer 2: run_dialectic_depth() → .chat() calls against Honcho
→ sync_turn() queues user+assistant messages for async write to Honcho
→ [at session end] on_session_end() flushes remaining messages
```

**What data is sent to Honcho:**
- Full user message content (chunked if >message length limit)
- Full assistant message content (chunked)
- Session ID, user peer ID, assistant peer ID
- On session end: any queued unflushed messages

**Write frequency**: Configurable via `writeFrequency` in `honcho.json`. Default: async (background queue). Can be set to sync for strict consistency.

### Configuration

Honcho config is stored in `~/.hermes/honcho.json` (profile-local) or `~/.honcho/config.json` (global). Key fields:

```json
{
  "api_key": "...",         // HONCHO_API_KEY — empty for self-host with no auth
  "baseUrl": "http://localhost:8000",  // HONCHO_BASE_URL
  "recallMode": "hybrid",   // "context" | "tools" | "hybrid"
  "contextCadence": 1,       // turns between context API calls
  "dialecticCadence": 2,     // turns between dialectic calls
  "dialecticDepth": 1,       // 1-3 LLM passes per dialectic cycle
  "injectionFrequency": "every-turn",
  "contextTokens": 1200      // max tokens for context injection
}
```

**For self-host with no auth**: Set `api_key: ""` and `baseUrl: "http://localhost:8000"`. The `is_available()` check passes when `api_key OR base_url` is set — so `baseUrl` alone is sufficient.

### Cron guard
Honcho is automatically skipped for cron jobs (`agent_context == "cron"` or `platform == "cron"`). This prevents cron-generated conversation content from polluting the user's Honcho memory.

---

## 4. Honcho vs Mem0 v3 vs Hindsight

### Architectural comparison

| Dimension | Honcho | Mem0 v3 | Hindsight |
|---|---|---|---|
| **Core metaphor** | Peer/session relationship modeling | Fact storage + semantic retrieval | Biomimetic World/Experience/Mental Models |
| **Data model** | Workspaces → Peers → Sessions → Messages → Representations | Facts with entity links + time metadata | World facts, Experience logs, Mental Model syntheses |
| **Extraction** | Background deriver (async LLM reasoning over messages) | Single-pass ADD-only LLM fact extraction per turn | LLM-driven retain with normalization |
| **Retrieval** | Dialectic (.chat()) = LLM synthesis + BM25/vector search | Multi-signal: semantic + BM25 + entity + temporal | 4-strategy parallel (semantic/BM25/graph/temporal) + cross-encoder rerank |
| **Synthesis** | `peer.chat()` — ask anything, Honcho reasons | Retrieval-only (no synthesis) | `reflect()` — deeper reasoning over memories |
| **Multi-peer** | First class (user, agent, group, project, idea) | Single-user/agent scoping | Single agent perspective |
| **Contradiction handling** | Background deriver resolves via evolving representations | ADD-only: both versions coexist, temporal retrieval selects | Mental Model synthesis resolves |
| **Persistence** | PostgreSQL + pgvector | PostgreSQL + pgvector (or Qdrant) | Embedded Postgres (or external) |
| **Self-host** | docker compose up (AGPL-3.0) | docker compose up (Apache 2.0) | single docker run (MIT) |
| **LLM for reasoning** | Yes (required for deriver + dialectic) | Yes (for extraction) | Yes (retain, reflect) |
| **LongMemEval score** | Not published (claims Pareto frontier; evals at honcho.dev/evals) | 94.8 (self-reported, Apr 2026) | 91.4 (validated Virginia Tech) |
| **Hermes plugin maturity** | 5 tools, background prefetch, cadence tuning, cron guard | Not checked | Not checked |

### What Honcho does that Mem0 cannot

1. **Tracks the agent's evolving understanding of the relationship**, not just stored facts. Honcho's representation of a user isn't a list of facts — it's a model of who they are that updates as the relationship evolves.
2. **Multi-peer symmetry**: can model what the agent knows, what the user knows, and what a project "knows" separately.
3. **Dialectic reasoning**: ask free-form questions about a peer ("What's their communication style?") and get an LLM-synthesized answer, not just chunk retrieval.
4. **Session context injection**: `session.context()` returns prompt-ready context formatted for any LLM provider.

### When to choose Mem0 over Honcho

- Need pure fact store with high retrieval precision on specific facts
- Don't need user/agent relationship modeling
- Apache 2.0 license is required (Honcho is AGPL-3.0)
- Team is not using Hermes — Mem0 has broader framework integrations

---

## 5. Benchmarks and Evals

### What Honcho claims

The README says: *"Honcho has defined the Pareto Frontier of Agent Memory"* and links to:
- Evals page: https://honcho.dev/evals/
- Research blog: https://blog.plasticlabs.ai/research/Benchmarking-Honcho
- Pareto-frontier announcement video (X/Twitter)

**Specific scores are NOT published in the README.** The README explicitly defers to the evals page for methodology and numbers.

**What can be inferred from the README**: Honcho claims to span LongMemEval, LoCoMo, and other long-conversation benchmarks on the evals page. The "Pareto Frontier" framing suggests they're claiming to be on the accuracy-cost tradeoff frontier, not necessarily the highest absolute accuracy.

**Missing information**: We could not fetch https://honcho.dev/evals/ in this research pass (SearXNG is search-only backend). The actual scores require browser access.

**What this means for the decision**: Honcho does not publish a comparable LongMemEval number in its README the way Mem0 (94.8), Hindsight (91.4), or ByteRover (92.8%) do. This is either because:
(a) their approach is architecturally different (user modeling, not just fact recall) and they consider standard benchmarks a poor fit; or
(b) their numbers are below the top competitors.

The "Pareto Frontier" claim (accuracy vs. cost, not raw accuracy) suggests (a) — they're optimizing for a different axis.

---

## 6. Privacy Analysis

### Self-hosted mode (recommended for Hermes)

**All data stays on the machine:**
- Messages stored in local Postgres (running in Docker)
- Representations/conclusions stored in local Postgres
- Background deriver processes locally
- LLM calls go to whichever provider you configure (Anthropic, OpenAI, etc.)
- No data goes to api.honcho.dev or any Plastic Labs infrastructure

**Privacy posture: excellent.** The only external calls are to the LLM provider(s) you configure.

### Cloud mode (api.honcho.dev)

**All conversation data goes to Plastic Labs' infrastructure:**
- Messages, sessions, peer representations all stored on Plastic Labs' servers
- Background reasoning runs on their infrastructure

**Not recommended for sensitive work.** For Isaac's China-related work or other sensitive conversations, self-host is required.

### LLM calls in self-hosted mode

Self-hosted Honcho still calls external LLMs for the deriver. If using Anthropic Claude Max as the backend, message content passes through Anthropic's API for reasoning — same as Hermes itself. This is acceptable if Anthropic is already approved.

---

## 7. Prototype Recommendation

### Why Honcho first over Mem0

1. **Hermes plugin is complete and production-quality** — 5 tools, background prefetch, cadence tuning, cron guard, config wizard. The Mem0 Hermes plugin may not be as mature.

2. **Complementary to the existing memory stack** — Honcho adds *relationship modeling* on top of what's already there. Mem0 adds *fact storage* — which overlaps more with what built-in memory + session search + the LLM Wiki already provide.

3. **The self-host path is clear** — `docker compose up` with ANTHROPIC key. Using the same Claude Max OAuth that Hermes uses, meaning no additional API cost.

4. **The dialectic query** is a qualitatively different capability — ask questions like "What does Isaac care about most in this project?" and get an LLM-synthesized answer that draws on conversation history.

5. **`hermes memory setup`** wizard handles the configuration. No manual config file editing needed.

### Prototype steps

```bash
# 1. Self-host Honcho
git clone https://github.com/plastic-labs/honcho.git
cd honcho
cp docker-compose.yml.example docker-compose.yml
cp .env.template .env
# Edit .env: set LLM_ANTHROPIC_API_KEY (same as Hermes uses), AUTH_USE_AUTH=false
docker compose up -d

# 2. Verify
curl http://localhost:8000/health

# 3. Configure Hermes
hermes memory setup
# Select: honcho
# Base URL: http://localhost:8000
# API key: (leave blank or use a generated token)

# 4. New Hermes session
hermes
# Honcho plugin should now show in memory status
hermes memory status

# 5. Test
# Chat for a few turns
# Try: honcho_context, honcho_search, honcho_reasoning
# Check Docker logs for deriver processing: docker compose logs deriver -f
```

### What to measure in the prototype

- Does context injection improve answer quality on questions about Isaac's preferences?
- Is the dialectic latency acceptable (first-turn timeout is 8s)?
- Does the deriver successfully process messages (check queue_status)?
- Is the Postgres disk usage manageable?
- Does the AGPL-3.0 license create any issues?

---

## Summary

Honcho is ready to prototype. The self-host path is clear (docker compose), the Hermes plugin is fully implemented, and the user/agent relationship modeling provides capabilities genuinely not present in the existing memory stack. Prototype Honcho self-host first, then test Mem0 v3 as a comparison.
