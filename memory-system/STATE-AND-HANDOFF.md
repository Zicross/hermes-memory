---
type: handoff
date: 2026-06-08
owner: Isaac / Zicross
subject: OpenClaw audit + Hermes memory research — state, decisions, and how to continue solo
status: research-complete; awaiting Isaac decisions before Phase 2 architecture
authored-by: Claude (Opus 4.8), driven from host zicrone with read-only access to the old OpenClaw install + this repo
fulfills: the 2026-06-08 memory-system-buildout handoff (OpenClaw cross-reference requirement + Phase 1 research)
---

# State & handoff: OpenClaw audit + Hermes memory research

Hand-off so the **Hermes agent can continue this work itself**, calling a Claude research subagent for
specific deep-dive threads. Read this top-to-bottom, then see "How to continue."

## 1. What this effort was

The memory-system-buildout handoff required cross-referencing the *previous OpenClaw install* before
choosing a memory architecture, and the prior agent could not locate it. We located it, audited it,
and then ran the Phase-1 provider/literature research. This file is the consolidated state.

## 2. Where OpenClaw actually is (the prior agent couldn't find it)

OpenClaw never lived in this container. It ran in the Windows-side **WSL "Ubuntu" distro**, a separate
disk image. On host `zicrone` (dual-boot; Windows partition mounted read-only):
- vhdx: `/mnt/windows/Users/isaac/AppData/Local/wsl/{25f56453-…}/ext4.vhdx` (78 GB).
- Mounted read-only on the host (`qemu-nbd` → `/mnt/wsl-ubuntu`) and the relevant layers bind-mounted
  into THIS container read-only: `/mnt/openclaw-engine`, `/mnt/openclaw-runtime`, `/mnt/openclaw`.
- A **secret-scrubbed copy** of OpenClaw agent state is in this container at `/root/openclaw-audit/state`
  (sessions + secrets removed). Full detail in `memory-system/openclaw-cross-reference.md` §Provenance.
- NOTE: the NBD mount is host-side and **not persistent across a host reboot**; re-mount steps are in
  the cross-reference doc. The Hermes agent (in-container) cannot create the mount itself — that needs
  host root (Isaac / a host Claude session).

## 3. Deliverables produced (all in `memory-system/`)

| File | What it is | Git |
|---|---|---|
| `openclaw-cross-reference.md` | OpenClaw memory audit; answers acceptance #4 | **committed + PUSHED** (5d6ba81) |
| `provider-comparison.md` | Phase 1 provider + literature comparison | committed local (579aa0f), not pushed |
| `production-patterns-and-recommendation.md` | What successful prod systems converge on; Hermes-native facts | committed local (579aa0f), not pushed |
| `scaling-and-multi-agent-memory.md` | Round 3: why files break at scale; the automatic-memory reframe | committed local (b69e15d), not pushed |
| `memory-approaches-primer.md` | Conceptual primer on all approaches + their philosophies (shared mental model) | this commit, not pushed |
| `STATE-AND-HANDOFF.md` | this file | this commit, not pushed |

Per Isaac: **committed locally, not pushed to GitHub yet.** Push when Isaac approves.
(Also note: the Hermes agent has its own parallel research piling up in `wiki/raw/articles/` —
reconcile, don't duplicate.)

## 4. Key findings (the load-bearing ones)

**OpenClaw (what to reject):** memory = PARA markdown repo + `qmd` per-agent SQLite. The "hybrid
semantic" layer **silently ran keyword-only** (all 790 chunk embeddings empty) — nothing flagged it.
Unsupervised nightly LLM "consolidation/self-improvement" crons (run twice nightly via duplicate
"backup" jobs) **rewrote the memory into word-salad** and deleted files; the index (`MEMORY.md`) drifted
to point at files that no longer exist. Lesson: **the store wasn't the problem — undisciplined writes +
unverified retrieval were.**

**What successful production systems converge on:** agent-first markdown **files** + agent-driven
retrieval + **compaction** + just-in-time loading. Managed vector stores are reserved for large external
corpora, not working memory. (Claude Code, Anthropic memory-tool, Cline, Cursor/Windsurf, Manus, Devin.)

**Hermes-native (verified):** Hermes = **NousResearch Hermes Agent**. `~/.hermes/bin/tirith` is its
command-security scanner, NOT the agent. Native memory is **two-tier**: always-on file memory
(`~/.hermes/memories/MEMORY.md` + `USER.md`, currently ~2 KB) PLUS one optional **pluggable provider**.
**Nine native providers:** Honcho, Mem0, Hindsight, OpenViking, Holographic, RetainDB, ByteRover,
Supermemory, Memori (`hermes memory setup`). The Hermes provider layer **wraps providers to be
automatic** (auto-recall before each turn, auto-retain after) — this is the key unlock.

**The scaling reframe (Isaac's concern, validated):** the file approach is **discretionary** (agent
must choose to write/read — already failing), rots as it grows, and corrupts under concurrent
multi-agent writes (= OpenClaw's clobbering). Fix = **automatic memory**, which Hermes provides by
enabling a provider. So **do not defer a provider** for the multi-agent/business path.

**Provider scorecard (for scale + multi-agent), deciding axis = tenancy/scoping + concurrency:**
- **Hindsight** — top pick to prototype (MIT, lightest self-host, bank-based scoping proven multi-agent,
  best-architected retrieval: vector+BM25+graph+temporal).
- **Honcho** — best tenancy model (Workspace/Peer/Session) + true precomputed auto-injection; but AGPL,
  heaviest, eventual-consistency (deriver queue), highest cost.
- **Memori** — dark horse (Apache, automatic by default, entity/process/session scoping, SQL-inspectable,
  mature ~15k★); weaker semantic recall.
- **Mem0** — mature/peer-reviewed but FLAT scoping (wrong shape for an org) + documents ~25% recall loss
  at 10M tokens.
- **Holographic** — DROPPED for backbone (HRR capacity ceiling, off-by-default, weak multi-agent); keep
  only as single-agent/air-gapped curiosity.
- **Caveat:** NO vendor's scale numbers are independently audited → benchmark on our own corpus.

## 5. Decisions made vs. still open

**Settled with Isaac:**
- Memory should be **agent-queryable, not human-formatted** — but the evidence-corrected reading is
  *agent-optimized files, NOT an opaque DB* (inspectability is how we'll catch the next silent failure).
- Do **not** mirror OpenClaw's PARA repo; do **not** salvage OpenClaw content (rebuild from Discord later).
- Cross-reference doc committed + pushed; the rest committed locally, not pushed.

**Open (awaiting Isaac):**
1. Accept the reframe — **adopt an automatic provider now** (vs. files-first)?
2. Prototype order **Hindsight → Honcho → Memori**?
3. Drop **Holographic** from backbone consideration?
4. Approve building the **benchmark harness** as the Phase-3 gate?

## 6. Recommended architecture (current best, pending decisions)

Two layers, each doing what it's good at:
- **Layer 1 — small curated inspectable files** (`SYSTEM_BOOT.md`, `model-policy.md`, identity/config):
  stable operational truth; the write-routing policy already in `model-policy.md` is good — keep it.
- **Layer 2 — one automatic Hermes provider** for episodic + semantic + shared/private memory, with
  auto read+write (kills the discretionary failure) and real tenancy + concurrency (survives multi-agent).
- **Hard gate (OpenClaw lesson):** a benchmark harness — recall / scale / stale-state / concurrency /
  liveness — before trusting any provider.

## 7. Remaining phases (from the buildout handoff)

- **Phase 2 — Architecture proposal** (`memory-system-architecture.md`): finalize the two-layer design,
  what-goes-where, retrieval order, write-routing, fresh-session boot path, backup/restore.
- **Phase 3 — Prototype + benchmark harness**: enable ONE provider (Hindsight first) on a tiny seeded
  set; build and run the harness.
- **Phase 4 — Verification/rollout**; **Phase 5 — maintenance loop** (idempotent, NOT unsupervised LLM
  rewriting — the OpenClaw anti-pattern).
- **Acceptance** (unchanged): a fresh Hermes session must answer the six acceptance questions with
  tool-backed verification, including #4 (OpenClaw lessons) and #5 (provider selected/deferred + why).

## 8. How to continue — driving solo + calling Claude as a research subagent

**Hermes drives**; delegate discrete *research* threads to a Claude subagent. What worked here and is
worth repeating:
- **Fan out** narrow, independent research tasks in parallel (one provider / one question each), then
  synthesize. Don't ask one agent to "research memory" broadly.
- **Prompt shape that worked** (reusable template):
  > "Research <X> as of 2026, WITH SOURCES, for a single-user→multi-agent self-hosted agent. Answer
  > precisely: (1) automatic write? (2) automatic read/injection? (3) retrieval at scale? (4)
  > multi-agent tenancy/concurrency? (5) cost/latency? (6) self-host/license. Be skeptical; separate
  > marketing from verifiable capability; flag self-reported benchmarks. Return ONLY structured markdown
  > with source URLs."
- **Good next research threads** (not yet done): hands-on Hindsight vs Honcho vs Memori self-host
  teardown (real configs, tenancy/concurrency internals); how Graphiti represents supersession; design
  of the benchmark harness; reconciling the Hermes agent's own `wiki/raw/articles/` findings with this.
- **What Claude could see:** read-only OpenClaw mount on the host + this repo. For deeper OpenClaw
  forensics (e.g., sanitized session transcripts), a host Claude session is needed (the mount is host-side).

## Related
- Buildout handoff: `handoffs/2026-06-08-memory-system-buildout-handoff.md`
- Primer (shared mental model): `memory-system/memory-approaches-primer.md`
- All analysis: `memory-system/*.md`
- Prior root-cause handoff: `/home/hermes/projects/civicbridge/planning/handoffs/2026-06-07-memory-system-handoff.md`
