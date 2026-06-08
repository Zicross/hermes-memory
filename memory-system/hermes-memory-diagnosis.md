# Hermes Memory Diagnosis — Why Recall Is Failing

**Date:** 2026-06-08  
**Status:** Confirmed root causes + fixes

---

## What's actually happening (verified from source)

Memory IS being injected into the system prompt. The system prompt contains a frozen snapshot of MEMORY.md (7 entries, 2,331 chars) and USER.md (5 entries, 1,491 chars), loaded at session start from `~/.hermes/memories/`.

**The problem is not that memory is absent — it's that it's passive.**

### The full mechanics

```
Session starts
→ MemoryStore.load_from_disk() reads MEMORY.md + USER.md → frozen snapshot
→ format_for_system_prompt() renders frozen snapshot into the volatile part of system prompt
→ System prompt = stable (identity/tools/skills) + context (SYSTEM_BOOT.md/SOUL.md) + volatile (memory/user/timestamp)
→ System prompt is CACHED for the whole session (prefix cache preservation)
→ Memory block appears in every turn from the start

BUT:
→ MEMORY_GUIDANCE tells Hermes HOW to write memory — not to actively recall it
→ No external provider = no automatic prefetch or injection of RETRIEVED memories
→ session_search is discretionary — Hermes must choose to call it
→ Memory entries are POINTERS to deeper context (SYSTEM_BOOT.md, shared repo)
   but Hermes doesn't automatically follow those pointers
```

### Why compression doesn't make it worse

Context compression DOES rebuild the system prompt (`invalidate_system_prompt()`), and it re-calls `load_from_disk()` which picks up any writes made during the session. So compression actually refreshes memory. This is not the problem.

### The two `skip_memory=True` cases

1. Memory review background agent (the hygiene sub-agent that occasionally runs)
2. Cron/flush agent contexts

Neither of these is the main chat session. The main gateway Discord session does NOT skip memory.

---

## Root cause table

| Root cause | Symptom | Severity |
|---|---|---|
| Memory entries are pointers, not content | Hermes sees "read SYSTEM_BOOT.md" but doesn't go read it | High |
| No external provider = no auto-prefetch | Relevant memories from past sessions aren't surfaced | High |
| Discretionary failure: session_search not used proactively | Historical context isn't recalled | High |
| MEMORY_GUIDANCE only teaches writing, not recall | Hermes knows how to save, doesn't know to fetch | Medium |
| MEMORY.md at 94% capacity | No room for new entries; stale pointers competing for space | Medium |
| USER.md entry 3 is stale: "defer memory buildout until core API/auth done" | Hermes might de-prioritize memory work when it shouldn't | Low |

---

## Fixes

### Fix 1 (immediate): Curate MEMORY.md

Free up space and fix stale entries. Changes:
- Remove the "defer memory buildout" entry from USER.md (stale — we're building it now)
- Consolidate the shared repo pointer into one clear entry
- Add an explicit "check session_search for recent decisions" instruction

### Fix 2 (immediate): Enable holographic provider as bridge

Holographic is a zero-dependency provider that ships natively in Hermes. It uses Holographic Reduced Representations (HRR algebraic memory) + trust scoring. No Docker, no API key, no external service. Not production-grade, but:
- Removes the discretionary failure immediately (auto read/write)
- Gives real data on what a provider integration feels like before standing up Hindsight
- Zero cost, zero privacy exposure

Enable: `hermes config set memory.provider holographic`

### Fix 3 (short-term): Enable Hindsight self-host

```bash
docker run -it --pull always --name hindsight --restart unless-stopped \
  -p 8888:8888 -p 9999:9999 \
  -e HINDSIGHT_API_LLM_API_KEY=$ANTHROPIC_API_KEY \
  -v hindsight-data:/home/hindsight/.pg0 \
  ghcr.io/vectorize-io/hindsight:latest
hermes memory setup   # select hindsight, point at http://localhost:8888
```

### Fix 4 (ongoing): Recall habits

Add explicit recall prompts to SYSTEM_BOOT.md so Hermes checks context at turn start:
- "Before answering questions about past decisions → use session_search"
- "Before assuming something hasn't been done → check hermes-memory/projects/"
- "Current model/provider facts → verify live with hermes status, not memory"

---

## What NOT to change

- Don't add more pointers to MEMORY.md — it's already full of them
- Don't remove SYSTEM_BOOT.md — it's working well as stable ops context
- Don't change compression threshold — it's fine and rebuilds memory on trigger
- Don't disable the frozen snapshot pattern — it's correct (preserves prefix cache)
