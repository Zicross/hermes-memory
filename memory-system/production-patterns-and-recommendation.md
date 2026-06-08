---
type: research
date: 2026-06-08
subject: Production memory patterns + Hermes-native capabilities → final direction (Round 2)
status: draft-for-Isaac-review
supersedes: section 0 of ./provider-comparison.md (Hermes-identity caveat now resolved)
related: ./provider-comparison.md, ./openclaw-cross-reference.md
---

# Production patterns + Hermes-native capabilities → recommended direction

Round 2: "what do successful agent systems actually do, especially considering Hermes."

## 1. Correction: what Hermes actually is

Hermes is **NousResearch's Hermes Agent** (docs: hermes-agent.nousresearch.com). Earlier I guessed
the `tirith` binary was the agent — **wrong**. `tirith` is Hermes's **command-security scanner**
(screens shell commands before exec; `tirith_enabled`/`tirith_path`/`tirith_fail_open` in
config.yaml). The strings in the binary are all command-interception logic, confirming this.

**Hermes native memory = two tiers, and it's pluggable:**
1. **Always-on file memory:** `~/.hermes/memories/MEMORY.md` (durable facts/learnings) +
   `USER.md` (user profile). Plain files. This is the spine and never turns off.
2. **One optional external provider** that injects context into the system prompt, prefetches
   before each turn, syncs after responses, extracts on session end, and **mirrors built-in memory
   writes**. **Nine providers ship natively:** Honcho, Mem0, Hindsight, OpenViking, Holographic
   (HRR algebra + trust scoring), RetainDB, ByteRover, Supermemory, Memori. Enable via
   `hermes memory setup` / `memory.provider:` in config.yaml.

**Consequence:** we don't build a store. Every option we evaluated is a config choice. The decision
shrinks to (a) how hard to lean on native file memory vs (b) whether/which native provider to add.

## 2. What successful production systems converge on

Strong, consistent convergence across the systems that actually work at scale:

- **Files/markdown are the durable memory substrate, read/written by the agent on demand.**
  Claude Code (`CLAUDE.md` + auto `MEMORY.md` + `memory` tool), Anthropic's memory-tool +
  context-editing beta, Cline "Memory Bank," Cursor/Windsurf rules + Memories, Manus
  ("filesystem as context… unlimited, persistent"), Devin (Knowledge + Playbooks).
- **Compaction + just-in-time retrieval are universal** for long runs: summarize, then re-hydrate
  from disk; hold lightweight references and load on demand rather than pre-stuffing context.
- **Vector/semantic stores are reserved for large external corpora**, not the agent's working memory.
  The managed-store camp (LangMem-style) is a real but minority position.
- **Context rot is measured and real** (Chroma, 18 models): accuracy drops monotonically as input
  grows, even below the window limit. Fewer high-signal tokens beat more tokens.
- **Reasons given are always the same:** files are inspectable, editable, unlimited, KV-cache-friendly,
  and avoid silent retrieval failures (exactly OpenClaw's failure mode).

The punchline: **the industry-winning pattern is essentially Hermes's own native design** —
agent-first files + compaction + optional semantic layer. "Doing it better" = using that pattern with
discipline, not bolting on machinery.

## 3. The "human-readable vs human-queryable" tension, resolved by evidence

Your steer — memory should be agent-queryable, not formatted for humans — is right, with one
correction the evidence forces: **the winning systems still use inspectable markdown files**
(`CLAUDE.md`, `MEMORY.md`). They are *agent-first* (terse, structured, atomic), not *human-styled*
(no narrative prose, no decorative formatting). So:

- "Not human-readable" should mean **"agent-optimized files,"** NOT "abandon files for an opaque DB."
- The evidence does **not** support a DB-only store; it supports the opposite. Going opaque would
  throw away the inspectability that lets you catch the next silent corruption (the OpenClaw lesson).
- Net: keep Hermes's `MEMORY.md`/`USER.md` as the spine; write them atomically and for retrieval;
  don't dress them up for a human reader. That satisfies your goal and the literature simultaneously.

## 4. Recommended direction (updated, Hermes-specific)

**Phase A — exhaust the native + convergent pattern first (build almost nothing):**
1. Treat `MEMORY.md` + `USER.md` + `SYSTEM_BOOT.md` + `model-policy.md` as the agent-first spine,
   maintained by **strict write-routing** (the policy already in `model-policy.md` is good — keep it).
2. **Compaction + just-in-time retrieval** for long sessions; `session_search` for historical detail.
3. **Append-only; no unsupervised LLM rewriting of canonical files** (OpenClaw's fatal flaw).
4. A **recitation/active-task file** for long autonomous runs (Manus pattern) to fight goal drift.
5. **Recall test** = the handoff's six acceptance questions, tool-verified in a fresh session.

This alone is what Claude Code / Manus / Devin run in production. Measure it before adding anything.

**Phase B — add ONE native provider only if Phase A's recall test fails** (specifically on semantic
or temporal "what's the current decision/state" queries). Selection constraint: **native ∩ free ∩
self-hostable ∩ privacy-safe**, with a verified recall test so it can't silently no-op like OpenClaw's qmd.

**Defer/avoid:** custom stores (Hermes already provides the substrate); heavy providers (Honcho —
user-modeling overkill for one user, heaviest to run); SaaS-only (Supermemory, RetainDB — fail privacy).

## 5. If we add a provider — shortlist (native AND free/local)

| Provider | Native? | License/host | Memory model | Why consider | Watch-out |
|---|---|---|---|---|---|
| **Hindsight** | yes | MIT, self-host | graph + entity resolution + temporal, multi-strategy recall | Best fit: free, local, adds the *temporal/entity* recall files lack; interpretable | newer project |
| **OpenViking** | yes | AGPL, self-host | filesystem-paradigm, tiered on-demand, token-efficient | Aligns with files-as-context; cuts tokens (good vs context rot) | AGPL if redistributed; ByteDance-origin (sensitive-work note) |
| **Mem0** | yes | Apache, self-host (Ollama+Qdrant) | extract→update consolidation loop | Borrow the contradiction-resolving write loop | weak multi-fact recall; LLM cost per write |

Recommendation if Phase B triggers: prototype **Hindsight** first (free, local, temporal/entity recall
is the capability files genuinely lack and that maps to Hermes's "forgot what was decided" pain).

## 6. Updated open decisions for Isaac
1. Agree to **Phase A first** (lean on native file memory + discipline + recall test; build nothing custom)?
2. If a provider is needed, prototype **Hindsight** as layer-2 (vs OpenViking / Mem0)?
3. Confirm the "agent-first files, not opaque DB" reading of your "not human-readable" steer — i.e. we keep MEMORY.md/USER.md, just agent-optimized?
4. Have the Hermes agent run `hermes memory setup` to confirm the live provider list + config shape before we finalize?
