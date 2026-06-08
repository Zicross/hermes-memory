---
type: research
date: 2026-06-08
subject: Memory provider comparison + architecture direction (Phase 1)
status: draft-for-Isaac-review
satisfies: handoff Phase 1 (memory-system-research.md); acceptance criteria #4, #5, #6
related: ./openclaw-cross-reference.md, ../handoffs/2026-06-08-memory-system-buildout-handoff.md
note: benchmark numbers below are largely vendor-reported and directional, not independently verified. Licenses, self-host availability, and the Hermes-native findings were cross-checked.
---

# Memory provider comparison & architecture direction

Research-first per the handoff. Goal: choose a memory architecture for Hermes — a single-user,
long-running, **headless** agent — where memory must be **agent-queryable, not human-readable**.
Self-hosting preferred; privacy matters.

## 0. What Hermes natively is (verified from the install)

- Hermes is a **compiled binary** (`~/.hermes/bin/tirith`, 12 MB). It is **not** verifiably the
  open-source NousResearch `hermes-agent`; do not assume its provider list.
- **Built-in memory = ~2 KB**: `~/.hermes/memories/MEMORY.md` (2,192 B) + `USER.md` (1,371 B).
  This is the "2,200-char block masquerading as a memory system" the 2026-06-07 CivicBridge
  handoff diagnosed. It is for *compact durable facts only* and is near capacity.
- `plugins/` is **empty**; no external memory provider is wired in. Session history lives in
  `sessions/` + `state.db` and is reachable via `session_search` (pull, not push).
- **Open action (only the Hermes agent can do this):** run `hermes memory setup` / `hermes memory
  status` and load the `hermes-memory-providers` skill (to be created if absent) to enumerate what
  provider interface `tirith` actually exposes. The architecture below is **substrate-agnostic** so
  it holds regardless of that answer.

## 1. The question you actually asked: what do Honcho & Mem0 do *uniquely*?

Neither does anything architecturally impossible. Both **package a cognitive pipeline** you would
otherwise build and maintain. The irreducible value is the *maintained pipeline + its tuning/evals* —
**not** the storage.

- **Mem0** — the unique piece is the **automatic write-side extract→update loop**: on each write an
  LLM decides ADD / UPDATE / DELETE / NOOP against existing memories, so contradictions get
  *reconciled over time* instead of piling up. That consolidation logic (and the prompts/evals behind
  it) is the one genuinely tedious thing to build well. Everything else (chunk, embed, vector/BM25
  search) is standard RAG. Cost: ~2 LLM calls **per write**. Apache-2.0, fully self-hostable offline
  (Ollama + Qdrant/pgvector). Weakness: multi-fact recall degrades hard (~61%→25% as evidence items
  grow); extraction is lossy/silent.
- **Honcho** — the unique piece is a **continuously-derived, evolving model of the user**: a
  background "Deriver" reasons over each message into a per-peer *Representation*, and a "Dream"
  process periodically re-derives new inferences. You query it in natural language (the "Dialectic"
  API — which is *just packaging*, replicable with your own prompt over retrieved chunks). The
  defensible edge is the maintained, auto-updating psychological/user model. For a **single user**,
  its multi-peer modeling is largely wasted, and it's the **heaviest to self-host** (Postgres+pgvector
  +Redis+worker loops + an ongoing background LLM bill even when idle). AGPL-3.0.

**Bottom line:** what you "can't easily do otherwise" = Mem0's contradiction-resolving write loop and
Honcho's evolving user-model. Both are *replicable*; the value is not rebuilding/maintaining them.
This restates the OpenClaw lesson exactly: **the hard part is write-discipline, not the store.**

## 2. What the literature actually supports (evidence, not vibes)

- **Pure context engineering** is the best-evidenced default **until history exceeds the window**
  (~150 conversations / ~100k tokens; ConvoMem). For a multi-year agent you *will* exceed it — so
  context engineering is necessary but **not sufficient**.
- **Managed semantic / temporal-KG memory** (Zep/Graphiti, HippoRAG): the *reproducible* benefit is
  **token/latency reduction and multi-hop/temporal recall**, not big accuracy wins. The genuinely
  differentiated capability is **temporal modeling** — knowing *when a fact was true and when it was
  superseded*. That maps directly onto Hermes's stated pain ("forgot/misremembered what was already
  decided") and onto OpenClaw's "stale-state" failure.
- **Filesystem beats bespoke libraries** (Letta benchmark): a plain *filesystem agent* (files + grep)
  scored **74% on LoCoMo, beating Mem0's graph variant (68.5%)**. Agents wield familiar tools (files,
  search) better than specialized memory APIs. **Strongest signal for a self-hosted single agent.**
- **The "LLM wiki" / auto-linking idea** (A-MEM): plausible and aligned with your instinct, but the
  benefit rests on a **single self-reported ablation on the weakest benchmark**, unreplicated. Treat
  auto-linking as an experiment to validate, not a settled win.
- **Benchmarks are shaky.** LoCoMo is near-saturated and partly mis-keyed (an audit found 6.4% of
  answers wrong and the judge accepting 63% of intentionally-wrong answers); the Mem0↔Zep numbers
  were publicly disputed/retracted. **Prefer LongMemEval; distrust LoCoMo rankings.**

## 3. On "human-queryable, not human-readable"

Your steer is directionally right, with a precision tweak from the evidence: markdown prose itself is
**not** harmful — agents read markdown natively and a link graph aids structural/multi-hop retrieval.
The real failure mode is **formatting *for humans* at the expense of retrievability** (long narrative
pages, decorative structure, implicit context a chunker can't index). So the rule isn't "avoid
markdown," it's **"optimize the store for retrieval, never for human presentation."** Atomic,
self-contained, embedded notes — queried by the agent — satisfy your goal whether the bytes on disk
happen to be markdown or rows in a DB. (And we are explicitly **not** mirroring OpenClaw's PARA repo.)

## 4. Provider landscape (condensed; full briefs in commit history)

**Self-hostable OSS, worth shortlisting**
- **Cognee** (Apache-2.0) — "memory control plane": `add → cognify → search` over pluggable graph
  (Neo4j/FalkorDB/NetworkX) + vector (Qdrant/pgvector) backends; best balance of license + self-host
  flexibility + graph/vector recall for one agent.
- **Hindsight** (MIT) — MCP memory server, entity resolution + multi-strategy recall (semantic/BM25/
  graph/temporal, reranked). Strong on the graph/entity features; interpretable.
- **Graphiti** (Apache-2.0, the engine under Zep) — **bitemporal** knowledge graph (`valid_from/
  valid_to/invalid_at`). Pick *only if* temporal supersession reasoning is a hard requirement; needs a
  graph DB (Neo4j/FalkorDB — note Zep CE is deprecated, Kuzu archived after Apple's acquisition).
- **Letta** (Apache-2.0, MemGPT successor) — tiered self-editing memory; but you adopt its *agent
  runtime*, not just a store. Consider only if unifying agent+memory.
- **Mem0** (Apache-2.0) — adopt for its consolidation loop *if* self-hosted with a local LLM; treat
  retrieval as weak on multi-fact.

**Defer / drop**
- **Honcho** — defer unless evolving user-modeling becomes an explicit goal; heaviest to run.
- **OpenViking** (AGPL) — interesting token-efficient filesystem paradigm; hold.
- **ByteRover** — only if Hermes becomes primarily a *coding* agent.
- **Supermemory, RetainDB** — **drop**: SaaS-first, no real free self-host path → fails privacy/self-host.

**Infra lanes:** embedded vector (**LanceDB** or **pgvector**) is the low-burden default; add a graph
DB only if adopting a graph-native layer. Avoid Kuzu (archived).

## 5. Recommended direction (for your decision)

A layered architecture where **the store is deliberately boring and the discipline is rigorous** —
the inverse of OpenClaw:

1. **Hot path = context engineering.** Boot context (`SYSTEM_BOOT.md`, `model-policy.md`) + built-in
   compact facts, assembled deterministically. Wins until scale; cheapest.
2. **Substrate = agent-managed atomic notes + a *verified* embedded index** (FTS5 + embeddings, or an
   embedded vector store). This is the Letta-filesystem-validated, lowest-risk baseline, self-hosted,
   queried by the agent. **Non-negotiable from OpenClaw: a liveness/recall test that proves embeddings
   actually exist and a semantic-only paraphrase query retrieves a seeded fact** (acceptance #6).
3. **Write-routing discipline** (the actual fix): classify every durable output per the handoff's
   policy; **append-only** notes; **no unsupervised LLM rewriting of canonical files** (OpenClaw's
   fatal flaw); any curator *proposes diffs*, never silently edits.
4. **Temporal layer — only if the baseline fails** on "what is the *current* decision/state" queries.
   If so, prototype **Graphiti** (bitemporal) — it targets Hermes's exact forgetting pain.
5. **Managed providers (Mem0/Honcho): not yet.** Borrow Mem0's *consolidation pattern* (the
   extract→update prompt) without adopting its stack. Reconsider a provider only after the baseline is
   measured and found wanting — never to avoid writing the policy.

**Why not jump to a provider:** every failure in OpenClaw was write-discipline + unverified retrieval,
not the store. A fancier provider does not fix that; policy + a recall test do.

## 6. Proposed Phase 3 prototype (smallest test that proves it)

- Seed ~20 known facts via the chosen substrate.
- Fresh-session recall test = the handoff's six acceptance questions, tool-verified.
- Adversarial stale-state test (assert it does NOT return a superseded fact as current).
- Deletion/update test.
- If recall on multi-hop/temporal questions fails → add Graphiti and re-run; else stop (baseline wins).

## Open decisions for Isaac
1. **Substrate:** agent-managed notes + verified embedded index (recommended) — OK as the baseline to prototype first?
2. **Temporal:** is "track when decisions/state were superseded" important enough to plan Graphiti as the likely layer-2 (vs. defer)?
3. **Providers:** agree to **defer Mem0/Honcho** (borrow Mem0's consolidation prompt only), or do you want a head-to-head Mem0-self-hosted bench in Phase 3 anyway?
4. Should I have the Hermes agent run `hermes memory setup`/`status` to enumerate `tirith`'s native provider interface before we finalize the substrate?
