---
type: research
date: 2026-06-08
subject: OpenClaw memory system — cross-reference audit for Hermes memory buildout
status: complete
satisfies: handoff "OpenClaw cross-reference requirement"; Phase 0 inventory (OpenClaw section); acceptance criterion #4
source: previous OpenClaw install, located + mounted read-only (see provenance)
---

# OpenClaw memory system — cross-reference audit

This fulfills the [memory-system-buildout handoff](../handoffs/2026-06-08-memory-system-buildout-handoff.md)
"OpenClaw cross-reference requirement" and answers acceptance criterion #4
("What was the OpenClaw installation and what lessons did we reuse or reject?").

## Provenance (where it was found / how accessed)

The handoff's quick search didn't find OpenClaw under `/home/hermes` — correct, because
**OpenClaw never lived in the Hermes container.** It ran inside the Windows-side **WSL "Ubuntu"
distro**, which is a separate `ext4.vhdx` disk image:

- Host: `zicrone` (dual-boot; Windows partition `/dev/nvme0n1p3` mounted **read-only** at `/mnt/windows`).
- Distro image: `/mnt/windows/Users/isaac/AppData/Local/wsl/{25f56453-…}/ext4.vhdx` (78 GB).
- Mounted read-only on the host via `qemu-nbd` → `/mnt/wsl-ubuntu`, then the OpenClaw layers
  bind-mounted into this container at `/mnt/openclaw-engine`, `/mnt/openclaw-runtime`, `/mnt/openclaw`.
  A secret-scrubbed copy of the agent state is at `/root/openclaw-audit/state`.
- The Windows `…/github/Projects/openclaw-studio` repo is a **stale (Apr 2) checkout**, not the runtime — ignore it as a source of truth.

Engine: `openclaw@2026.4.5` (npm global). Runtime instance: `/home/openclaw-agent/openclaw`.
Agent state: `/home/openclaw-agent/.openclaw`. **Canonical agent workspace:**
`/home/openclaw-agent/.openclaw/workspace`. Active period: ~Mar 19 – Apr 10 2026 (then idle).

> Note: OpenClaw and Hermes ran **as peers**, not pure succession — OpenClaw's `MEMORY.md`
> lists Hermes as an active independent agent with a `/shared/tasks/` file-drop protocol.

## TL;DR — reuse / reject

**Reuse (the good ideas):**
- Layered split: human-readable PARA knowledge repo + a separate semantic index + session memory. Maps cleanly onto the handoff's target layers.
- A local/self-hosted semantic store (SQLite + FTS5, optional local embeddings) — no external API dependency. Right instinct for privacy.
- Privacy scoping: `MEMORY.md` loaded **only in main/direct sessions**, never in shared/Discord; semantic scope **deny-by-default**.
- Pre-compaction "memory flush" that writes durable notes to dated daily files, with canonical boot files (`SOUL/AGENTS/MEMORY`) treated as **read-only** during the flush.

**Reject (what actually broke):**
- **Unsupervised, LLM-driven nightly rewriting of memory.** This is the single biggest lesson. It corrupted the knowledge base.
- Trusting a "hybrid semantic" config that **silently degraded to keyword-only** with no health signal.
- "Backup" cron jobs that **re-run the same generative rewrite** 30 min later (doubles drift, not redundancy).
- Convention-by-prompt (file-naming rules in a prompt) with no mechanical enforcement.

---

## Architecture as-built (the handoff's required inspection points)

### 1. Persistence store
Two parallel stores:
- **Human-readable PARA workspace** — `~/.openclaw/workspace/`, markdown, indexed by a hand/auto-maintained `MEMORY.md`. Structure: `projects/`, `areas/`, `resources/`, `skills/`, `memory/` (dated daily notes), `archive/`. This is the "Knowledge Map."
- **`qmd` semantic index** — per-agent SQLite at `~/.openclaw/memory/{main,amazon-ceo,software-ceo}.sqlite`. `main.sqlite` = 4.3 MB, **126 files / 790 chunks** (688 from memory, 102 from sessions). Schema: `files`, `chunks(text, embedding, …)`, `chunks_fts` (FTS5), `embedding_cache`, `meta`. Chunking: 400 tokens, 80 overlap, `unicode61` tokenizer.

### 2. Retrieval mechanism
- **Configured** (`openclaw.json` → `agents.defaults.memorySearch`): hybrid **vector 0.7 / text 0.3**, MMR (λ=0.7), temporal decay (30-day half-life), `maxResults` 8, `maxSnippetChars` 800, sources `[memory, sessions]`.
- **Actual** (from `main.sqlite`): `meta.model = "fts-only"`, `provider = "none"`, `embedding_cache` = **0 rows**, and **all 790 chunk embeddings are 2 bytes (`[]`)**. → The vector half **never ran**. In practice retrieval was **BM25 keyword search only**. The sophistication was aspirational; nothing surfaced the gap.

### 3. Write policy
- **Pre-compaction memory flush hook** (`compaction.memoryFlush`, soft threshold 10k tokens): prompt instructs the agent to append durable memories to `memory/YYYY-MM-DD.md`, treat `MEMORY.md/SOUL.md/TOOLS.md/AGENTS.md` as read-only, **append-only**, no timestamped variant files. (Good intent.)
- **Two autonomous nightly cron skills**, both `always:true`, `disable-model-invocation:true`, **no human in the loop**:
  - `nightly-consolidation` — reads daily notes, "extracts durable info," **promotes to PARA**, **summarizes/rewrites old notes**, archives, re-indexes.
  - `self-improvement` — reviews transcripts and is explicitly licensed to **"modify memory files and skills freely."**
- **Schedule (`cron.json`):** consolidation `0 2` **+ duplicate backup `30 2`**; self-improvement `0 3` **+ duplicate backup `30 3`** — and the same four again for the `amazon-ceo` agent. So **~4 generative rewrite passes per night** over the same files.

### 4. Sync mechanism
- `qmd` watch + incremental: `update.interval` 5m, debounce 15s, `onBoot` true, `embedInterval` 60m. Live file-watching of the markdown collections.
- No external sync for the runtime workspace; the separate `openclaw-studio` git repo was a manual/parallel mirror (and drifted).

### 5. Prompt-injection strategy
- Semantic results injected via `memorySearch` (top-8, 800-char snippets), **scope deny-by-default** with allow-rules only for `direct` chats and `discord`.
- `MEMORY.md` loaded into context **only in main session**, never shared contexts (explicit security boundary). Daily notes (today+yesterday) read at session startup per `AGENTS.md`.

### 6. Provider / API dependencies
- Embeddings were meant to be **local** (`~/.cache/qmd/models`) → no external embedding API. Web search via Brave; models via Gemini-CLI OAuth + GitHub Copilot (Anthropic disabled at the time). For *memory* specifically: **zero external provider** — but the local embedding path was effectively inert (see retrieval).

---

## Lessons / pitfalls (evidence-backed)

### Pitfall 1 — Autonomous LLM rewriting degrades memory into nonsense
The PARA layer **rotted**. `MEMORY.md` (last updated Apr 5) advertises six `areas/` files plus
several `resources/`. Reality on disk:
- **Missing:** `tacit-knowledge.md`, `work-journal.md`, `system-knowledge.md`,
  `performance-tracking.md`, `resources/credentials.md`, `resources/research-index.md`.
- **Surviving but corrupted:** `areas/failure-registry.md` (231 B) is literal word-salad —
  *"Pain-based task-reactivity, where a 'fail-on-anomaly log' forget IF uncaught in detection
  pattern-level override STOP harmful-grepped SHIFT for Why relied mas retain mini undescribed."*
  `areas/lessons-learned.md` (323 B) trails off mid-sentence into garble.
- Corruption timestamps (failure-registry **Apr 11**, lessons-learned **Apr 13**) are **after**
  the agent otherwise went idle (~Apr 10) — i.e., the nightly crons kept "consolidating" the
  files into mush on the last nights. Strong (inferred) root cause: repeated unsupervised LLM
  summarize-in-place with no validation and no human review.

**For Hermes:** treat agent-written knowledge as **append-only**; promotion/curation must be
**human-reviewed or mechanically validated**, never a free nightly LLM rewrite of canonical files.
If a curator runs, it should *propose* diffs (like OpenClaw's boot-file path) — not silently edit.
Reconciles with the handoff write-routing policy and Phase 5 maintenance loop.

### Pitfall 2 — Silent semantic-search failure
The config claimed hybrid vector retrieval; the index ran `fts-only` with empty embeddings and
**nothing flagged it**. A memory system can look sophisticated and be keyword-only in production.

**For Hermes:** acceptance criterion #6 must include a **liveness check that proves the semantic
layer actually embeds and recalls** (e.g., seed a known fact, confirm a semantic-only paraphrase
query retrieves it, assert embedding dims > 0). Don't trust config; test recall.

### Pitfall 3 — "Backup" jobs that duplicate generative work
The `*-backup` crons re-ran the same rewrite 30 min later. Redundancy for a **generative**
operation isn't safety — it's double the drift.

**For Hermes:** maintenance jobs must be **idempotent** (re-running changes nothing if healthy).

### Pitfall 4 — Conventions enforced only by prompt
The flush prompt forbade `…-HHMM.md` variant files; the `memory/` dir is full of them
(`2026-04-06-1814.md`, `2026-04-09-2341.md`, topic-suffixed notes, etc.).

**For Hermes:** enforce naming/format mechanically (the writer tool/skill names the file), not by asking the model nicely.

### Pitfall 5 — Index/source-of-truth drift
`MEMORY.md` confidently points at files that no longer exist — itself a form of the
"hallucinate stale state" failure the handoff warns about. An index without a link-checker rots silently.

**For Hermes:** the boot/index doc should be **validated against the filesystem** (dead-link check) on a cadence; surface drift loudly.

---

## qmd vs the handoff's provider candidates

`qmd` is essentially the **self-hosted SQLite + FTS5 (+ intended local embeddings)** option from
the handoff's list (comparable lane to LanceDB/Chroma/pgvector but lighter, file-watching, markdown-native).

Implications for the Phase 1 comparison:
- The predecessor's self-hosted baseline is **real and low-burden** — a hosted provider (Honcho/Mem0/Supermemory) must justify added dependency/privacy cost against "SQLite+FTS over the `hermes-memory` repo, with verified embeddings."
- The decisive variable wasn't the *store* — FTS5 worked fine. It was **write discipline** and **verified retrieval**. A fancier provider does **not** fix Pitfall 1; write-routing policy does. (Matches the handoff's "do not treat a vector DB as a magic replacement for write policy.")

## Direct answer — acceptance criterion #4

**What was the OpenClaw installation?** A Discord-resident autonomous agent ("The Big Z") on
`openclaw@2026.4.5` in WSL, with a two-part memory system: a PARA markdown knowledge repo (indexed
by `MEMORY.md`) plus a per-agent `qmd` SQLite semantic index over memory files + session transcripts,
maintained by pre-compaction flushes and nightly consolidation/self-improvement crons.

**Lessons reused:** layered memory (human-readable + semantic + session), self-hosted/local store,
privacy scoping (main-session-only sensitive memory, deny-by-default semantic scope), append-on-flush
to dated notes with read-only boot files.

**Lessons rejected:** unsupervised LLM rewriting/summarizing of canonical memory (corrupted it),
trusting unverified "hybrid" retrieval (silently keyword-only), duplicate generative "backup" crons,
and prompt-only convention enforcement.

## Open questions for Isaac
1. Should the `hermes-memory` git repo be the human-readable PARA layer (replacing OpenClaw's `workspace/`)? It already mirrors the shape.
2. Reuse the qmd *approach* (SQLite+FTS+local embeddings over the repo) as Hermes' self-hosted semantic baseline, or evaluate Honcho/Mem0 against it from scratch?
3. Is any OpenClaw memory *content* worth salvaging (e.g., `resources/playbooks/`, `discord-channels.md`, the freelance project knowledge), or is this purely an architecture lesson and the content is abandoned?

---
*Audit evidence (config rows, embedding lengths, file inventory, cron schedule) captured from the
read-only mount; secret-scrubbed state copy at `/root/openclaw-audit/state`. Raw session transcripts
remain only in the read-only host mount, not copied into this container.*
