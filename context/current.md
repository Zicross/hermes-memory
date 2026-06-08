# Current Context

**Last updated:** June 8, 2026

## Active Projects

- **ConstiuINT** — CivicBridge MVP in progress. See [[projects/constiuint]]
- **Hermes memory-system buildout** — research/architecture phase. See:
  - `handoffs/2026-06-08-memory-system-buildout-handoff.md`
  - `research/2026-06-08-hermes-memory-system-research-and-architecture.md`
  - `memory-system/architecture.md`
  - `memory-system/llm-wiki-integration.md`

## Current Workflow

User (Claude Opus on laptop) → orchestrates → Gemma 4 26b (laptop) → implements
Opus → reviews Gemma's output
Hermes → strategic oversight, cross-project coordination, memory/source-of-truth maintenance

## Verified Hermes Runtime Snapshot

Verified June 8, 2026:

- Primary Hermes model: `openai-codex / gpt-5.5`
- Fallback: direct `anthropic / claude-sonnet-4-6` only
- External memory provider: none; Hermes built-in memory only
- OpenRouter: active for verified-free auxiliary models, not fallback
- Source-of-truth files: `~/.hermes/config.yaml`, `~/.hermes/SYSTEM_BOOT.md`, `~/.hermes/model-policy.md`, this repo, and relevant skills

Always re-verify live config before changing model/provider/memory settings.

## Recent Decisions

- Gemma 4 already running on laptop (not something to set up)
- Hermes no longer does day-to-day coding orchestration
- Cross-project memory system is being built as a layered architecture
- External memory provider selection is deferred; prototype Honcho self-host first and RetainDB Local second
- LLM Wiki should be part of the learning/research layer for memory-system buildout

## What's Changed Since Last Check

- June 7: Created Claude Code memory base
- June 7: Updated ConstiuINT AGENTS.md to new workflow
- June 8: Added memory-system handoff, OpenClaw audit, provider research, architecture proposal, and LLM Wiki integration plan

## Next Steps

1. Build the LLM Wiki slice for memory-provider research
2. Run recall test suite in a fresh Hermes session
3. Decide whether to start Honcho self-host prototype
4. Continue ConstiuINT work in Opus/Gemma workflow
