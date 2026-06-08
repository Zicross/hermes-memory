# LLM Wiki Integration for Hermes Brain

**Status:** Add to architecture as the research/learning knowledge compiler layer.

## Why it matters

Karpathy's LLM Wiki pattern is a strong fit for Isaac’s goal: learn as much as possible while building the best possible memory system. It differs from generic RAG/external memory:

- RAG retrieves chunks at query time.
- A wiki compiles knowledge once into interlinked pages.
- Contradictions, source provenance, summaries, and cross-links are maintained explicitly.
- The user can read, edit, and learn from the same artifact the agent uses.

This is exactly the missing layer between raw research/session transcripts and semantic memory providers.

## Proposed role in Hermes architecture

Add an **LLM Wiki / compiled knowledge** layer inside or alongside `/home/hermes/claude-code-memory/`:

```text
/home/hermes/claude-code-memory/wiki/
├── SCHEMA.md
├── index.md
├── log.md
├── raw/
│   ├── articles/
│   ├── papers/
│   ├── transcripts/
│   └── assets/
├── entities/
├── concepts/
├── comparisons/
└── queries/
```

For the memory-system project, likely wiki pages:

- `concepts/agent-memory.md`
- `concepts/semantic-memory.md`
- `concepts/hybrid-retrieval.md`
- `concepts/memory-consolidation.md`
- `entities/honcho.md`
- `entities/retaindb.md`
- `entities/hindsight.md`
- `entities/mem0.md`
- `entities/supermemory.md`
- `entities/openviking.md`
- `entities/byterover.md`
- `comparisons/external-memory-providers.md`
- `comparisons/vector-vs-graph-vs-wiki-memory.md`

## Authority policy

The wiki should be authoritative for **compiled research synthesis**, not live runtime state.

- Current Hermes config still comes from live tools/config.
- Memory architecture decisions come from `memory-system/` and `decisions/log.md`.
- Provider facts in the wiki must carry source/provenance and updated dates.
- External semantic memory can index/query the wiki, but should not replace it.

## Workflow

1. Ingest raw sources into `wiki/raw/` with source URL, date, and hash.
2. Compile/update entity/concept/comparison pages with wikilinks.
3. Update `wiki/index.md` and `wiki/log.md` every time.
4. Use the wiki for learning, provider comparisons, and contradiction tracking.
5. Promote settled implementation decisions into `memory-system/` and `decisions/log.md`.

## Prototype recommendation

Before enabling Honcho/RetainDB as external semantic memory, build a small LLM Wiki slice for the memory-provider research. This gives Isaac a readable learning artifact and gives Hermes a durable compiled knowledge base to query later.

Minimum viable LLM Wiki prototype:

1. Create `wiki/SCHEMA.md`, `wiki/index.md`, `wiki/log.md`.
2. Add raw source snapshots for Honcho, RetainDB, Hindsight, Mem0, Supermemory, OpenViking, ByteRover, and OpenClaw memory docs.
3. Create one comparison page: `comparisons/external-memory-providers.md`.
4. Create concept page: `concepts/layered-agent-memory.md`.
5. Run a lint pass for broken links/index completeness.

## Why this improves provider selection

It prevents provider choice from being a one-off chat answer. The system builds a durable map of:

- what each provider claims,
- what Hermes currently supports,
- what OpenClaw already taught us,
- what contradictions/risks exist,
- what experiments are needed,
- which decision was finally made and why.
