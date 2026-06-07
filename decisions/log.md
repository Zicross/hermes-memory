# Decisions Log

Settled decisions that should NOT be forgotten. Check here before asking "what did we settle on?"

## Model Stack (June 2026)

| Use Case | Model | Cost |
|----------|-------|------|
| Sensitive (China) | Claude Pro + Gemma 4 26b | $20/mo |
| Everyday Hermes | MiniMax M2.5 | Free |
| Server local | Qwen3.5-8b | Free |
| Coding | Opus → Gemma 4 | $20/mo + laptop |
| Review | Codex | $20/mo |

**Key:** Gemma 4 already running on laptop via Ollama/Tailscale. NOT something to set up.

## Workflow (June 2026)

- **Old:** Hermes → Claude Code → Codex (HERMES DID THE BUILDING)
- **New:** User (Opus) → Gemma 4 → Opus reviews (USER DOES THE BUILDING)
- **Hermes:** Strategic direction, cross-project, oversight, NOT day-to-day coding

## Cross-Project Memory

- Hermes maintains cross-project brief in Obsidian
- Each project has rich AGENTS.md
- Claude Code memory base: `/home/hermes/claude-code-memory/` (this folder)

## Mobile-First

ConstiuINT is MOBILE-FIRST (phone app), NOT desktop-first. This was missed initially — future projects should capture mobile-first in initial requirements.

---

*When we settle on something new, add it here immediately.*