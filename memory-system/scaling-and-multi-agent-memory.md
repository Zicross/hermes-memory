---
type: research
date: 2026-06-08
subject: Scaling & multi-agent memory — does "Phase A" break? (Round 3)
status: draft-for-Isaac-review
supersedes: the "Phase A first, build nothing" framing in ./production-patterns-and-recommendation.md
related: ./provider-comparison.md, ./production-patterns-and-recommendation.md, ./openclaw-cross-reference.md
---

# Does the file-memory approach break at scale + multi-agent? (Yes — here's the fix)

Isaac's concern: the file/context approach (Phase A) eventually gets too big, the agent stops
choosing to remember/recall (already happening), and it will break as we move to business workloads
with multiple agents. **This is correct and well-evidenced. The fix changes the architecture.**

## 1. Why it breaks (the discretionary-memory failure)

File/context memory (CLAUDE.md, MEMORY.md, Memory Bank, Anthropic's memory tool) is **discretionary
and unbounded** — and fails on three fronts:

1. **The agent must *choose* to write and *choose* to read.** Both are unreliable. Anthropic's memory
   tool ships an ALL-CAPS "ALWAYS VIEW YOUR MEMORY DIRECTORY BEFORE DOING ANYTHING" system prompt —
   they shout because the model otherwise skips it. Practitioners report the exact two-sided failure
   (no write → nothing to recall; no read → on-disk memory stays invisible). **This is Isaac's symptom,
   and it's expected behavior, not a misconfiguration.**
2. **Growth → context rot.** Load-everything degrades recall (Chroma's 18-model study: accuracy drops
   monotonically as input grows; lost-in-the-middle >30% drop). Load-on-demand puts you back in the
   discretionary trap. Core bind: *load-everything rots, load-on-demand forgets.*
3. **Multi-agent → concurrency corruption.** Flat files have no transactions: parallel agents cause
   lost updates, dirty reads, and contamination cascades. **OpenClaw already demonstrated this** (config
   clobbered 11×, memory files corrupted to gibberish). Files cannot express scoping or merge policy.

## 2. The fix: automatic memory — and Hermes already provides it

The fix removes the model's discretion on both sides:
- **Automatic write** — a background pipeline (not the agent) extracts and persists what matters.
- **Automatic read** — relevant memories are injected into context every turn with no tool call.

**Key Hermes-specific finding:** the Hermes provider layer *wraps* its memory providers to be automatic
— per the NousResearch docs, an active provider "automatically recalls memories before each turn" and
"automatically retains conversation turns" after responses, even for providers whose raw library is
call-based (Mem0, Hindsight). **So enabling a native Hermes provider directly fixes the
"agent doesn't choose to remember" problem** — that is the single biggest reason to adopt one, and it
argues against deferring it (the Round-2 "Phase A first" stance) for the multi-agent business path.

Tradeoff to accept (honestly): automatic memory costs an LLM call per write (latency + tokens) and
will store some wrong/ephemeral "facts" (false memories). That's the price of reliability at scale.

## 3. What actually matters at multi-agent business scale

Ranked by what will make-or-break a fleet of agents doing business work:
1. **Tenancy / scoping** — private-per-agent + shared-team + read-only "core" knowledge zones.
   This is the axis files completely lack and where providers differ most.
2. **Concurrency / consistency** — deterministic merge over last-write-wins; isolation for shared state.
3. **Automatic read+write** — non-negotiable given the discretionary failure (Hermes supplies this).
4. **Retrieval that holds as the corpus grows** — and here's the sobering part: **nobody has solved it.**
   Every vendor's scale numbers are self-reported and often internally inconsistent; Mem0 is the only one
   that publishes an honest scaling tell (BEAM: ~25% recall loss from 1M→10M tokens). Treat all scale
   claims as unproven until benchmarked on our own corpus.

## 4. Vendor scorecard on the scale/multi-agent axes (Hermes natives)

| Provider | Auto W/R (in Hermes) | Tenancy / multi-agent | Retrieval @ scale | Self-host | Verdict |
|---|---|---|---|---|---|
| **Hindsight** | yes / yes | **bank-based** (per-user + shared-company; proven multi-agent in a fintech case) | best-*architected* (vector+BM25+graph+temporal, reranked) but numbers self-reported/inconsistent | **MIT, lightest** (1 Docker cmd, embedded PG) | **Top pick to prototype** |
| **Honcho** | yes / **yes (precomputed inject)** | **best tenancy** (Workspace→Peer→Session, true shared-vs-private) | least-evidenced; **eventual consistency** (deriver queue) risky under bursty load | AGPL, heaviest (PG+Redis+workers) | Strong if tenancy is paramount + cost/AGPL OK |
| **Memori** | **yes by default** | real **entity/process/session** scoping | SQL/FTS (no vector-grade ANN); inspectable as SQL rows | Apache, mature (15k★), easy | Dark horse: auditable, DB-native |
| **Mem0** | yes / yes | **flat** (user_id/agent_id) — wrong shape for an org | peer-reviewed but documents ~25% loss at 10M | Apache, easy | Mature but thin tenancy |
| **Holographic** | off by default | weak (profile-isolated only) | HRR **capacity ceiling**; no benchmarks | first-party, zero-dep | **Drop for backbone**; single-agent/air-gapped only |

(OpenViking: tiered L0/L1/L2 loading = real scale lever, profile-only. ByteRover: pre-compaction
extraction, code-centric. Supermemory: best multi-container sharing but SaaS. RetainDB: best retrieval
stack but no auto-write + SaaS.)

## 5. Revised architecture (automatic from the start for the business path)

Two layers, each doing what it's good at:
- **Layer 1 — small, curated, inspectable files** (`SYSTEM_BOOT.md`, `model-policy.md`, identity/config).
  Stable operational truth. This is what Claude Code etc. keep; it does NOT carry episodic recall.
- **Layer 2 — one automatic Hermes provider** for episodic + semantic + shared/private memory, with
  auto read+write (kills the discretionary failure) and real tenancy + concurrency (survives multi-agent).

This deliberately does NOT wait for files to break — because they're already breaking and the multi-agent
plan guarantees the other two failure modes.

## 6. The catch: scale is unsolved industry-wide → mandatory benchmark harness

No vendor's large-corpus recall is independently audited; the best (Hindsight) has inconsistent
self-reported numbers. So the OpenClaw lesson hardens into a requirement: **before committing to any
provider, run a recall + scale + concurrency harness on our own data:**
- Recall test = the handoff's six acceptance questions, fresh session, tool-verified.
- Scale test = seed N×10 memories, confirm recall doesn't collapse (watch the multi-fact falloff).
- Stale-state test = a superseded fact is NOT returned as current.
- Concurrency test = two agents writing the shared scope don't lose/corrupt each other's writes.
- Liveness = embeddings/derivations actually populate (the OpenClaw fts-only silent-failure check).

## 7. Open decisions for Isaac
1. Accept the reframe: **adopt an automatic provider now** (not "files first"), because the discretionary
   failure + multi-agent plan make Phase-A-alone a known dead end?
2. Prototype order: **Hindsight first** (MIT, lightest, multi-agent banks, best-architected retrieval),
   with **Honcho** as the tenancy-rich alternative and **Memori** as the inspectable-SQL dark horse?
3. Drop **Holographic** from backbone consideration (keep only as a single-agent/air-gapped curiosity)?
4. Approve building the **benchmark harness** as Phase 3's gate before any provider is trusted?
