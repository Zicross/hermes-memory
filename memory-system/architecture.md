# Hermes Memory System Architecture

**Status:** Proposed; base markdown/source-of-truth layer approved for buildout; external provider deferred pending prototype.

## Layers

1. **Live state** — authoritative for current facts: `hermes status`, `hermes config`, `hermes memory status`, filesystem, git, cron list.
2. **Config** — `~/.hermes/config.yaml`; secrets only in `~/.hermes/.env`.
3. **Boot/policy docs** — `~/.hermes/SYSTEM_BOOT.md`, `~/.hermes/model-policy.md`.
4. **Built-in memory** — compact durable facts and pointers only; never architecture prose.
5. **Session search** — historical transcript recall and provenance.
6. **Skills** — reusable procedures, commands, pitfalls, verification recipes.
7. **Shared markdown repo / Obsidian-style knowledge base** — curated decisions, architecture, handoffs, research, project context.
8. **LLM Wiki / compiled knowledge layer** — raw sources plus interlinked entity/concept/comparison pages for learning-heavy research, provider comparisons, provenance, and contradiction tracking.
9. **External semantic memory** — optional associative/user/agent memory after base loop is reliable.
10. **Cron/maintenance** — review, hygiene, recall tests, provider health.

## Authority Order

- Current runtime state: live tools beat all notes.
- Model/provider routing: `config.yaml` + `SYSTEM_BOOT.md` + `model-policy.md` + `model-management` skill.
- Procedures: skills.
- Architecture/decisions: this repo + local planning docs.
- User preferences: current user instruction, then durable memory.
- History: session search.
- Semantic provider: hints only unless corroborated.

## OpenClaw Lessons Adopted

- Markdown remains source of truth.
- Always-loaded memory is an index, not a warehouse.
- Raw daily/session logs need promotion/classification.
- Hybrid lexical + vector retrieval is better than pure vector search for config/path facts.
- Evergreen policies should not decay like daily notes.
- Avoid split-brain brain roots, plaintext secrets, and config self-edit corruption.

## External Provider Stance

Production provider is deferred. Prototype order:

1. Honcho self-host — first, because Hermes supports it and it models people/agents/sessions well.
2. RetainDB Local — second, because it is promising for coding/project context and local-first operation.
3. Hindsight/Mem0/Supermemory later if the first two fail or if their strengths become necessary.

Do not enable multiple external providers at once except in isolated evaluation.
