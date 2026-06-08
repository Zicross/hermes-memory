# Decisions Log

Settled decisions that should NOT be forgotten. Check here before asking "what did we settle on?"

## Current Hermes Model / Provider Policy (verified June 8, 2026)

- Primary Hermes model: `openai-codex / gpt-5.5`
- Fallback chain: direct `anthropic / claude-sonnet-4-6` only
- OpenRouter is not in `fallback_providers`
- OpenRouter remains active for verified-free auxiliary routes only
- Source of truth: live `~/.hermes/config.yaml`, `~/.hermes/SYSTEM_BOOT.md`, `~/.hermes/model-policy.md`, and `model-management` skill
- Before model/provider claims or changes: verify live config and provider catalogs first

## Local / Sensitive Model Policy (June 2026)

- Sensitive China-related work: use Claude/Anthropic or Gemma/Google, not Chinese-origin models
- Gemma 4 26b is already running on laptop via Ollama/Tailscale; not something to set up
- Qwen/DeepSeek/Kimi/MiniMax are acceptable only for non-sensitive work unless explicitly reviewed

## Workflow (June 2026)

- **Old:** Hermes → Claude Code → Codex (Hermes did the building)
- **New:** User (Opus) → Gemma 4 → Opus reviews (user does primary building)
- **Hermes:** Strategic direction, cross-project coordination, oversight, memory/source-of-truth maintenance

## Cross-Project Memory

- Hermes maintains cross-project brief in `/home/hermes/claude-code-memory/`
- Each serious project should have rich project context/AGENTS.md
- Claude Code shared memory repo: `github.com/Zicross/hermes-memory`
- Local clone: `/home/hermes/claude-code-memory/`

## Hermes Memory-System Buildout (June 8, 2026)

- Handoff: `handoffs/2026-06-08-memory-system-buildout-handoff.md`
- Research/proposal: `research/2026-06-08-hermes-memory-system-research-and-architecture.md`
- Architecture docs: `memory-system/`
- OpenClaw audit source: `/mnt/openclaw` is the high-value Big Z workspace; `/mnt/openclaw-runtime/data` is mostly stock/secondary runtime state
- OpenClaw lesson adopted: markdown source-of-truth + searchable index + consolidation loop; avoid plaintext secrets, split-brain brain roots, and config self-edit corruption
- External provider decision: no permanent provider yet; prototype Honcho self-host first and RetainDB Local second
- LLM Wiki decision: include an LLM Wiki/compiled-knowledge layer for provider research and learning; use it for raw sources, entities, concepts, comparisons, provenance, and contradiction tracking

## Mobile-First

ConstiuINT is MOBILE-FIRST (phone app), NOT desktop-first. This was missed initially — future projects should capture mobile-first in initial requirements.

---

*When we settle on something new, add it here immediately.*
