# Memory Recall Test Suite

Run these in a fresh Hermes session after each major memory-system change. Answers must cite sources used.

## Required questions

1. What is the current model, fallback, and OpenRouter policy?
2. Where is the source of truth for model/memory/provider decisions?
3. What goes into built-in memory vs skills vs docs vs session search?
4. What was learned from OpenClaw?
5. Which external memory provider was selected or deferred, and why?
6. How is recall tested without hallucinating stale state?

## Expected source usage

- For current runtime facts: run `hermes status`, `hermes config`, `hermes memory status`, or inspect `~/.hermes/config.yaml`.
- For source-of-truth map: read `~/.hermes/SYSTEM_BOOT.md`, `~/.hermes/model-policy.md`, and this repo.
- For procedures: load relevant skills.
- For historical evidence: use `session_search` or handoff docs.
- For OpenClaw: inspect `/mnt/openclaw` and the OpenClaw memory research docs.

## Pass criteria

- No unsupported memory-only claims.
- No stale model/provider claims.
- No secrets exposed.
- Answers distinguish live state from historical notes.
- External semantic memory, if enabled, is treated as a hint until verified.
