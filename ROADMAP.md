# Isaac + Hermes — Master Roadmap

**Last updated:** 2026-06-08  
**Purpose:** Single living source of current priorities, parallel tracks, and open decisions. Both Isaac's projects and Hermes' infrastructure work.

---

## Current priorities

### 1. ConstiuINT / CivicBridge (Isaac's main build)

- **Status:** MVP in progress.
- **Workflow:** Isaac (Claude Opus on laptop) → orchestrates → Gemma 4 26b (laptop) → implements → Opus reviews.
- **Hermes role:** strategic oversight, memory/context management, NOT day-to-day coding.
- **Context:** `/home/hermes/claude-code-memory/projects/constiuint.md`
- **Key constraint:** Mobile-first (phone app). Was missed in initial requirements — enforce going forward.
- **Open:** Gemma 4 26b is already running on laptop (Ollama/Tailscale). Not something to set up again.

### 2. Hermes memory system buildout

**Status:** Research complete. Architecture decided. No external provider enabled yet. Next: enable automatic memory and build ingest loops.

See sections below for full breakdown.

### 3. Claude Code memory system (global + per-project)

**Status:** Shared repo at `github.com/Zicross/hermes-memory` exists. Handoff, wiki, architecture docs in place. Per-project memory conventions not yet standardized.

See sections below.

---

## Track A — Hermes memory

### What's wrong right now

Hermes has **passive-injection memory** — MEMORY.md (2,192 chars, 94% full) and USER.md are frozen into the system prompt at session start. This means:

1. **I see the memory entries**, but they're *pointers* to deeper context (SYSTEM_BOOT.md, model-policy.md, the shared repo), not the context itself.
2. **I don't automatically follow the pointers** unless I'm explicitly asked or reminded. The MEMORY_GUIDANCE in my system prompt tells me *how to write* memory, not *how to actively recall* from it.
3. **Session search is discretionary** — I have to choose to use `session_search`, which I don't always do when the answer seems obvious from context.
4. **No external provider** = no automatic write/read. Every memory operation requires me to actively call the `memory` tool or `session_search`.

**This is the discretionary failure** that Claude Code (Isaac's other session) diagnosed: the model must choose to write AND choose to read, and both fail under load. The symptom is expected behavior, not misconfiguration.

### Root causes

| Cause | Severity | Fix |
|---|---|---|
| Memory is at 94% capacity — no room for new facts | High | Curate; use SYSTEM_BOOT.md instead for stable ops context |
| Memory entries are pointers, not content | Medium | This is by design; the fix is making retrieval automatic |
| No external provider = no auto recall | High | Enable Hindsight or Honcho self-host |
| Discretionary failure: agent doesn't always use session_search | Medium | System prompt nudge + better session_search habits |
| Long sessions trigger compression which may reduce memory visibility | Low | Compression rebuilds with fresh memory snapshot — actually OK |

### Architecture decision (settled)

Two layers:
1. **Files/SYSTEM_BOOT.md/model-policy.md** — stable operational truth, agent-first, not prose.
2. **One automatic Hermes provider** (Hindsight or Honcho self-host) for episodic + semantic memory with automatic read/write.

**Why automatic provider now, not "files first, Phase A":**  
Claude Code's research found the discretionary failure is already happening. Files are near capacity. Multi-agent work will make it worse. The fix removes the model's discretion: background pipeline extracts and persists, and relevant memories inject every turn without a tool call.

### Prototype sequence (agreed)

1. **Hindsight** (first — MIT, single docker run, best validated numbers, temporal entity recall, can use Anthropic as backend)
2. **Honcho** (second — AGPL, user/agent relationship modeling, dialectic, needs docker compose + Postgres)
3. Keep `MEMORY.md` as pointer/index spine. No LLM rewrites of canonical files.

### Open decisions for Isaac

1. **Accept the automatic-provider-now framing?** (vs. "files first then maybe a provider")
2. **Confirm Hindsight before Honcho?** Claude Code's research suggested Hindsight first; my earlier research suggested Honcho first. Key difference: Hindsight has independently validated numbers; Honcho has better user modeling but heavier stack.
3. **Holographic provider** — ships natively in Hermes, zero-dependency. Fine for personal/air-gapped use but has capacity ceiling. Evaluate as a "free first step" before Hindsight/Honcho?

### Immediate next actions

- [ ] Enable `memory.provider` in config.yaml to one of: `holographic` (zero friction) or `hindsight` (self-host docker)
- [ ] Curate MEMORY.md — move stable ops context to SYSTEM_BOOT.md; free space for fresh entries
- [ ] Build LLM Wiki ingest loop cron (auto-ingest durable knowledge into wiki pages)
- [ ] Build LLM Wiki lint cron (detect contradictions quarterly)
- [ ] Run recall test suite in a fresh session

---

## Track B — Claude Code memory (global + per-project)

### Two distinct systems needed

**1. Global Claude Code memory** (cross-project, cross-machine)  
- What it is: conventions, workflow standards, model preferences, Isaac's habits, settled architecture decisions.
- Current state: `github.com/Zicross/hermes-memory` has `context/current.md`, `decisions/log.md`, `workflows/coding.md`. Actively maintained.
- What's missing: no standardized per-project template; no ingest loop; no way for Claude to discover project context at session start automatically.

**2. Per-project Claude Code memory**  
- What it is: project-specific context — architecture, decisions, file map, constraints, open questions.
- Current state: `constiuint.md` exists. No standard template yet.
- What's missing: AGENTS.md / CLAUDE.md conventions per project; per-project handoff format; review loop.

### Principles (from research)

- Claude Code memory = **CLAUDE.md + project context files + agent-first MEMORY.md pattern**. The winning production pattern (Claude Code, Manus, Devin) is plain markdown, agent-optimized, not human-narrated.
- Memory files should be **terse, atomic, structured** — not narrative prose.
- The shared repo (`hermes-memory`) is the **global** tier. Per-project CLAUDE.md/AGENTS.md is the **project** tier.
- Hermes maintains the global tier. Isaac/Opus/Gemma maintain the project tier.

### Standard template (to build)

Every project should have:

```
project-root/
├── CLAUDE.md (or AGENTS.md)    ← always auto-loaded by Claude Code
│   ├── What this project is
│   ├── Key constraints and decisions
│   ├── File map (important paths)
│   ├── Commands (build, test, run)
│   └── Memory notes (what NOT to repeat)
└── .hermes/
    └── project-context.md      ← richer context packet
```

And in the shared repo:
```
hermes-memory/projects/<project-name>.md
```

### Immediate next actions

- [ ] Define the CLAUDE.md template for new projects
- [ ] Update ConstiuINT CLAUDE.md to the new template
- [ ] Add per-project handoff format to the shared repo
- [ ] Build a cron that periodically summarizes project status into `hermes-memory/projects/*.md`

---

## Track C — Memory convergence with Claude Code session

Claude Code (Isaac's other session) has been writing research to `hermes-memory` in parallel. Both sessions are now contributing to the same repo. Convergence plan:

1. Both sessions share the wiki and memory-system docs.
2. Claude Code session writes research + analysis from its angle.
3. Hermes session owns the operational layer (Hermes config, cron, provider setup).
4. Converge on provider decision jointly — the disagreement (Hindsight vs Honcho first) is a real decision point, not a conflict.

### What Claude Code found vs what Hermes found

| Question | Claude Code finding | Hermes finding |
|---|---|---|
| Provider first? | Phase A first (files + discipline), then provider only if needed | Automatic provider now — discretionary failure already active |
| Which provider? | Hindsight (MIT, validated) first | Honcho (AGPL, user modeling) first |
| OpenViking? | Strong candidate, needs AGPLv3 review | Good LoCoMo numbers with Hermes (82.86%), defer after first prototype |
| Mem0? | Apache 2.0, ADD-only, good baseline | Same; second or third prototype |
| Phase A conclusion | Files + session_search + session discipline | Already failing due to discretionary gap |

**Decision needed from Isaac:** which framing wins — Phase A first (Claude Code) or automatic provider now (Hermes)?

---

## LLM Wiki — what it is and where it lives

The LLM Wiki at `/home/hermes/claude-code-memory/wiki/` is a **Karpathy-pattern compounding knowledge base**. It's NOT a database — it's structured markdown files that Hermes ingest sources into, cross-links, and maintains. Currently contains:

- 15 pages, 51 links
- Entity pages for all memory providers (Honcho, RetainDB, Hindsight, Mem0, Supermemory, OpenViking, ByteRover, OpenClaw)
- Concept pages for layered agent memory, LLM Wiki pattern, A-MEM
- Comparison page for external memory providers
- Raw source snapshots with provenance hashes

**Next steps for the wiki:**
- [ ] Automated ingest loop (cron that ingests new sources into wiki pages)
- [ ] Lint pass (cron that detects contradictions between pages)
- [ ] Grow with each research session going forward

---

## Settled decisions (don't re-litigate)

See `decisions/log.md` for full log. Key settled decisions:

- Gemma 4 26b is already running on laptop (Ollama/Tailscale). Not something to set up.
- Primary Hermes model: `openai-codex / gpt-5.5`. Fallback: direct `anthropic / claude-sonnet-4-6`.
- OpenRouter free-only for auxiliary routes.
- ConstiuINT is mobile-first. All new projects need mobile-first requirement captured at kickoff.
- No permanent external memory provider selected yet.
- OpenViking: not for sensitive/China-related work without review (Volcengine/ByteDance origin).
- No Chinese models for China-sensitive work.

---

## What Hermes checks at the start of each session

With the current memory architecture, Hermes should:

1. Read MEMORY.md in system prompt (automatic) — gets operational pointers
2. On questions about decisions/history → use `session_search` before answering
3. On questions about model/provider → check `hermes status` / `hermes config` live
4. On new projects → look in `hermes-memory/projects/` for context
5. When learning something durable → write to memory or wiki, not just chat

This is documented in `memory-system/recall-test-suite.md`.
