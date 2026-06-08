---
title: AI Agent Long-Term Memory Landscape Research 2026
created: 2026-06-08
type: research-report
tags: [agent-memory, architecture, benchmark, karpathy, hindsight, mem0, supermemory, openviking, a-mem, graphiti, cognee, longmemeval, locomo]
sources:
  - raw/articles/hindsight-readme.md
  - raw/articles/mem0-readme.md
  - raw/articles/supermemory-readme.md
  - raw/articles/openviking-readme.md
  - https://arxiv.org/abs/2502.12110 (A-MEM)
  - https://arxiv.org/abs/2512.12818 (Hindsight paper)
  - https://arxiv.org/abs/2504.19413 (Mem0 paper)
  - https://arxiv.org/abs/2603.29194 (Multi-layered memory)
  - web search (June 2026)
confidence: high
---

# AI Agent Long-Term Memory Landscape — Deep Research Report (June 2026)

This report covers the full breadth of current approaches to agent long-term memory, benchmarks, and concrete options for Hermes architecture. All sections include architecture takeaways.

---

## 1. The Karpathy LLM Wiki Pattern

### What It Is

Andrej Karpathy's LLM Wiki (gist `442a6bf555914893e9891c11537de94f`, published ~April 2026) describes a **pattern** — not a product — for building a personal knowledge base from plain Markdown files that an AI agent actively builds and maintains over time.

The core idea: instead of querying ephemeral documents at chat time (RAG), the agent *digests* sources into a growing, edited, human-readable wiki. The wiki itself becomes the permanent knowledge store, and it compounds with every new ingest.

### Exact Mechanics

The operational loop has three named operations:

1. **Ingest** — Given a new source (URL, PDF, conversation, document), an LLM agent reads it and either creates a new wiki article or amends an existing one. The content is distilled into clear, editable Markdown. Relationships between topics are made explicit with links. Metadata is attached (source URL, date, confidence).

2. **Query** — At runtime, the wiki index (a structured Markdown manifest listing article titles and one-line summaries) is loaded into context. The agent can then either read the full article or pass the index to find the most relevant entry. Named queries — reusable prompts that target specific sections — allow structured retrieval (e.g., "what is the current decision on X?").

3. **Lint** — A background pass runs contradiction detection: if two articles disagree, the lint operation flags or merges them. This quality gate is what keeps the wiki from drifting into inconsistency over time.

An optional **Status** operation adds a `status: draft|reviewed|outdated` frontmatter tag for knowledge lifecycle management.

The filesystem layout is typically:
```
wiki/
├── index.md          ← manifest of all articles
├── config.py         ← paths, LLM config
├── articles/
│   ├── topic-a.md
│   └── topic-b.md
└── prompts/          ← ingest, query, lint, status prompt templates
```

### Why It Beats RAG for Durable Knowledge

| Dimension | RAG | LLM Wiki |
|---|---|---|
| Representation | Raw document chunks | Distilled, human-editable prose |
| Updates | Re-embed the source | Edit one file; re-lint |
| Contradictions | Silently co-exist | Flagged by lint pass |
| Inspectability | Black box | Human-readable, git-diff-able |
| Compounding | None — same answers forever | Each new ingest enriches the entire KB |
| Context budget | Semantic search overhead | Index fits in context; full article on demand |

RAG retrieves *document fragments* — the same chunks for everyone, every time. The Wiki *understands* the content and restructures it. A query against the wiki hits a pre-digested synthesis, not raw text.

The "compounding principle": when a question gets answered well, the wiki learns that answer too. Useful insights stay; noise doesn't get ingested.

### How It Composes with Agent Loops

The LLM Wiki is **synchronous knowledge** that the agent reads before acting. In a typical agent loop:

```
1. Boot: load wiki index into system prompt
2. On task: query wiki for relevant articles → inject into context
3. After task: if new durable knowledge found → run ingest → update wiki
4. Periodic: cron fires lint pass → detect contradictions
```

It is NOT a retrieval system — it's a read/write knowledge layer. This makes it different from RAG (read-only fragments), different from Mem0 (fact tuples in a vector store), and different from graph memory (entity-relationship store). It is closest to a **maintained reference document** that lives in the agent's working environment.

### Limitations

- **Scale cap**: works well for hundreds to low-thousands of articles; degrades when the index grows beyond ~100KB (can't fit it all in context).
- **Ingestion cost**: each new source requires an LLM call to distill — not cheap at volume.
- **No temporal reasoning**: doesn't natively handle "this fact changed at time T"; lint can catch contradictions but not timelines.
- **Single-agent assumption**: the original pattern is for a personal wiki; multi-agent concurrent writes require locking or conflict resolution.

### Hermes Relevance

**The Hermes wiki at `/home/hermes/claude-code-memory/wiki/` IS already this pattern.** The existing index.md, entity articles, concept articles, and comparison articles implement the LLM Wiki. The next evolution is to formalize the ingest/lint loops and grow the compounding library. This layer should remain the **human-readable source of truth** layer in the Hermes memory stack.

---

## 2. Hindsight (Vectorize.io)

### What It Is

Hindsight is an **open-source agent memory system** built by Vectorize.io, MIT-licensed. It is focused on making agents *learn* from experience, not just recall conversation history. The name references the paper: *"Hindsight is 20/20: Building Agent Memory that Retains, Recalls, and Reflects"* (arXiv:2512.12818, Dec 2025).

### Architecture: Biomimetic Data Structures

Hindsight organizes memory into three types, modeled on human cognition:

- **World** — Objective facts about the world ("The stove gets hot")
- **Experiences** — The agent's own history ("I touched the stove and it really hurt")  
- **Mental Models** — Synthesized understanding formed by reflecting on World + Experiences

Each memory is stored as a combination of: entities, relationships, time series, sparse vectors (BM25), and dense vectors (semantic embeddings).

### The Three Operations

**Retain** (write):
- LLM extracts key facts, temporal data, entities, and relationships from the input
- Runs normalization: extracts canonical entities, indexes time series, creates search indexes
- Stores in the appropriate pathway (World or Experiences)

**Recall** (read): Runs **4 retrieval strategies in parallel**:
1. Semantic (dense vector similarity)
2. Keyword (BM25 exact matching)  
3. Graph (entity/temporal/causal links)
4. Temporal (time range filtering)
Results are merged via **reciprocal rank fusion** then reranked with a **cross-encoder model**.

**Reflect** (synthesize):
- Deeper reasoning pass over existing memories
- Generates new insights and connections (builds Mental Models)
- Used for complex analysis like "why did outreach messages fail?"

### Benchmark Performance

As of early 2026 (LongMemEval benchmark):
- **Hindsight + Gemini-3 Pro: 91.4% overall** (highest open-domain score 95.12%)
- Hindsight + OSS-120B: 89% overall
- Independently reproduced by Virginia Tech Sanghani Center and The Washington Post

**Caveats on benchmark claims:**
- LongMemEval is the 500-question S-setting (single-session seeding with multi-session recall)
- Hindsight scores are self-reported in the benchmarks repo; Virginia Tech reproduction confirms the number is not fabricated
- Mem0 v3 (April 2026) later claimed 94.8 on LongMemEval with a different methodology (single-pass ADD-only, OpenAI backbone)
- Mastra Observational Memory claims 94.87% with gpt-5-mini; OMEGA claims 95.4%
- **These numbers are not directly comparable** — different backbone LLMs (Gemini vs GPT-4o vs gpt-5-mini) explain most of the gap; the memory architecture is only part of the score

### Self-Host Docker Setup

Minimal (all-in-one):
```bash
export OPENAI_API_KEY=sk-xxx
docker run -it --pull always --name hindsight --restart unless-stopped \
  -p 8888:8888 -p 9999:9999 \
  -e HINDSIGHT_API_LLM_API_KEY=$OPENAI_API_KEY \
  -v hindsight-data:/home/hindsight/.pg0 \
  ghcr.io/vectorize-io/hindsight:latest
```

**Storage**: Embedded PostgreSQL (`/.pg0` volume) by default. External PostgreSQL available via docker-compose for production.

**LLM providers**: `openai`, `anthropic`, `gemini`, `groq`, `ollama`, `lmstudio`, `minimax` — set via `HINDSIGHT_API_LLM_PROVIDER`. Supports local/Ollama for full air-gap.

**Python embedded** (no server): `pip install hindsight-all` — runs everything in-process.

**API**: HTTP REST on port 8888. UI on port 9999.

**Oracle AI Database** supported for enterprise deployments.

### How It Differs from RAG and Knowledge Graphs

| Dimension | RAG | Knowledge Graph | Hindsight |
|---|---|---|---|
| Storage | Document chunks + vectors | Entity-relationship nodes | World/Experience/Model pathways |
| Retrieval | Vector similarity | Graph traversal | 4-strategy parallel + reranking |
| Learning | None | Append-only edges | Reflect synthesizes Mental Models |
| Temporal | None | Limited | First-class time-series indexing |
| Contradiction handling | None | Manual | Mental Model synthesis |

Hindsight explicitly rejects the flat-RAG model: it structures memory like human cognition rather than like a filing cabinet.

### Hermes Fit

- **Strong candidate** for the external semantic memory layer
- Docker-based, Anthropic-compatible (can use Claude as backbone)
- Can use Ollama for fully local operation
- The `reflect` operation aligns well with Hermes's periodic review concept
- **Concern**: heavier than needed for current scale (~750 lines); overhead may exceed benefit until memory volume grows
- **Prototype recommendation**: test after the core Markdown/wiki layers are stable

---

## 3. Mem0 (Open Source)

### What It Is

Mem0 ("mem-zero") is an open-source, Apache 2.0 memory layer for AI agents. YC S24 company. The paper is arXiv:2504.19413 (ECAI 2025): *"Mem0: Building Production-Ready AI Agents with Scalable Long-Term Memory."* GitHub: `mem0ai/mem0`.

### Memory Algorithm (v3, April 2026)

The April 2026 v3 algorithm is a major architectural shift:

**Key changes:**
1. **Single-pass ADD-only extraction** — One LLM call per conversation turn. No UPDATE or DELETE operations. Memories accumulate; old facts are never overwritten, only superseded by newer ones.
2. **Agent-generated facts are first-class** — When an agent confirms an action ("I scheduled the meeting"), that fact is stored with equal weight to user-stated facts.
3. **Entity linking** — Entities are extracted, embedded, and linked across memories for retrieval boosting.
4. **Multi-signal retrieval** — Semantic (dense), BM25 (keyword), and entity matching scored in parallel and fused.
5. **Temporal reasoning** — Time-aware retrieval ranks the right dated instance for queries about current state, past events, and upcoming plans.

**Why ADD-only?** The prior algorithm used UPDATE/DELETE operations (in-place fact mutation). This introduced complexity: if fact A contradicts fact B, which wins? The v3 answer: keep both, let retrieval return the most recent/relevant. This reduces write complexity dramatically and is more robust to LLM extraction errors.

### Benchmark Scores (v3)

| Benchmark | Old Algorithm | New (v3) | Tokens | Latency p50 |
|---|---|---|---|---|
| LoCoMo | 71.4 | **91.6** | 7.0K | 0.88s |
| LongMemEval | 67.8 | **94.8** | 6.8K | 1.09s |
| BEAM (1M) | — | **64.1** | 6.7K | 1.00s |
| BEAM (10M) | — | **48.6** | 6.9K | 1.05s |

**BEAM** is a new production-scale benchmark at 1M–10M token corpora (Mem0's own benchmark). BEAM at 1M: 64.1; at 10M: 48.6.

**Important caveat**: Mem0's scores are self-reported on their own evaluation framework. The open-source eval code is at `mem0ai/memory-benchmarks`. The LongMemEval 94.8 uses a different (stronger) backbone model than Hindsight's 91.4. Not directly comparable.

### Self-Hosted Docker Stack

```bash
cd server && make bootstrap    # one command: starts stack + creates admin + issues first API key
# or manually:
cd server && docker compose up -d   # → http://localhost:3000
```

**Stack**: FastAPI (REST API) + PostgreSQL with pgvector + optional Qdrant. Auth is ON by default.

**LLM requirement**: Default `gpt-5-mini` (OpenAI). Configurable to any LiteLLM-supported provider including Ollama.

**Embeddings**: Default `text-embedding-3-small`. For best results: at minimum Qwen 600M or `gte-Qwen2-1.5B-instruct`.

**Deployment options**:
- `pip install mem0ai` — library mode, no server
- `docker compose up` — self-hosted server with dashboard
- `app.mem0.ai` — managed cloud

### Memory Algorithm: Fact Extraction, Deduplication, Contradiction Handling

**Fact extraction**: LLM call on each message batch. Extracts factual assertions, preferences, and confirmed actions. Each fact becomes an independent memory unit.

**Deduplication (v3 ADD-only)**: In v3, deduplication is handled at retrieval time, not write time. The retrieval layer uses entity linking + recency signals to surface the most relevant (usually most recent) version of a fact.

**Contradiction handling**: In v3, contradictions are NOT resolved at write time. Both facts exist. The temporal retrieval layer surfaces the more recent fact for present-tense queries. Past-tense queries can still surface the older fact appropriately. This is a deliberate tradeoff: simpler writes, more retrieval complexity.

**Graph memory** (optional): An optional Neo4j/FalkorDB/Kuzu backend adds entity-relationship storage, giving ~2% accuracy improvement over flat storage.

### Hermes Fit

- **Strong candidate** for the external semantic memory layer
- Excellent ecosystem: LangChain, LangGraph, CrewAI, Mastra, OpenAI Agents SDK integrations
- ADD-only v3 algorithm is simple and robust; good for prototyping
- Self-hosted stack is straightforward
- **Concern**: v3 scores are self-reported, backbone-dependent
- **Prototype recommendation**: Strong first candidate — well-documented, broad community, open eval framework

---

## 4. LongMemEval and LoCoMo Benchmarks

### LongMemEval (arXiv:2410.10813)

**What it tests**: 500 manually created questions designed to evaluate a chat assistant's long-term memory across multi-session conversations.

**Five core memory abilities tested**:
1. **Information extraction** — Can the assistant recall a specific fact mentioned earlier?
2. **Multi-session reasoning** — Can it reason across information from different sessions?
3. **Temporal reasoning** — Does it understand time order? ("What did I say *before* I moved to SF?")
4. **Knowledge updates** — Does it correctly apply the most recent fact when facts contradict over time?
5. **Abstention** — Does it correctly say "I don't know" instead of hallucinating?

**Settings**: S (single-session seeding, multi-session recall) and M (multi-session seeding). Most public numbers use the S setting (easier).

**Scoring**: LLM-as-judge (typically GPT-4o). Each question is scored correct/incorrect. Reported as % correct.

**Known issues / caveats**:
- The judge model matters enormously. A GPT-5-mini judge vs GPT-4o judge produces different scores for the same answers.
- Backbone LLM matters as much as memory architecture. Hindsight with Gemini-3 scores higher than with OSS-120B; this isn't about the memory system, it's about the LLM's reasoning ability.
- The 500 questions are public — systems *can* be optimized for them specifically.
- **Providers don't all use the same evaluation setup**, making cross-vendor comparison unreliable.

**Current leaderboard (as of June 2026, claimed)**:
- OMEGA: ~95.4% (GPT-4.1 backbone)
- Mastra Observational Memory: 94.87% (gpt-5-mini)
- Mem0 v3: 94.8 (backbone unspecified, likely GPT-4o class)
- Hindsight: 91.4% (Gemini-3 Pro), 89% (OSS-120B)
- Supermemory: 81.6% (backbone unspecified)
- Zep/Memobase/LangMem: 30–60% range

**What the number actually means for Hermes**: LongMemEval tests whether *conversational facts* are accurately recalled and reasoned over. It's the right benchmark for user preference learning and session-to-session memory. It is NOT a test of knowledge-base retrieval, procedural memory, or planning-loop memory.

### LoCoMo (arXiv:2402.17753, Snap Research)

**What it tests**: 1,540 questions derived from very long-term synthetic conversations (up to 35 sessions, 300+ turns). Four question types:
1. **Single-hop** — Direct fact recall
2. **Multi-hop** — Requires combining facts from multiple sessions
3. **Temporal** — Time-order reasoning
4. **Adversarial** — Designed to trick with near-miss facts

**Key difference from LongMemEval**: LoCoMo tests much longer conversation histories (35 sessions vs ~5), emphasizing scalability of memory systems.

**Current scores (claimed)**:
- Mem0 v3: **91.6** (LoCoMo, April 2026, +20pp over prior)
- Supermemory: **#1** (no specific % published)
- OpenViking + Hermes: **82.86%** (with 66% latency reduction vs native)
- OpenViking + OpenClaw: **82.08%**
- Hindsight: ~70-80% range (not prominently self-reported)

**LoCoMo vs LongMemEval**: LoCoMo is harder (longer histories, adversarial questions); a system that scores well on LoCoMo likely has robust multi-hop temporal reasoning.

### ConvoMem (Salesforce)

Tests personalization and preference learning in conversations. Supermemory claims #1 on ConvoMem; Mem0 has weak reported scores (30–45% per independent MemPalace benchmark).

### What Scores Actually Matter for Hermes

For Hermes's use case (single long-lived agent, one user, durable facts about Isaac + projects):
- **LongMemEval** matters: tests the core capability of fact retention and temporal update
- **LoCoMo** matters at scale: if Hermes grows to 35+ session histories
- **BEAM** matters for scale: if memory grows to millions of tokens
- **ConvoMem** matters for personalization depth

**Hermes should weight**: LongMemEval (primary), LoCoMo (secondary), self-reported scores should be treated as upper bounds with the same backbone.

---

## 5. Key Academic Papers on Agent Memory

### A. Mem0: arXiv:2504.19413 (ECAI 2025)

**Title**: "Mem0: Building Production-Ready AI Agents with Scalable Long-Term Memory"  
**Authors**: Chhikara et al. (Mem0.ai team)

**Key contributions**:
- First broad head-to-head comparison of 10 memory approaches
- Mem0 achieves 26% relative improvement vs OpenAI's full-context method on LLM-as-judge metric
- Graph memory adds ~2% accuracy over flat storage
- Demonstrates token efficiency: 90%+ token reduction vs full-context while matching or exceeding accuracy

### B. A-MEM: arXiv:2502.12110 (NeurIPS 2025)

**Title**: "A-MEM: Agentic Memory for LLM Agents"  
**Authors**: AGI Research team  
**GitHub**: `agiresearch/A-mem`

**What it is**: A Zettelkasten-inspired memory system where each memory note has:
- A unique ID (like a Zettelkasten slip)
- Contextual description (automatically generated)
- Keywords and tags
- Links to related notes (auto-discovered)
- Evolution history (notes are updated, not replaced)

**Key innovation**: Memories are "agentic" — they generate their own connections. When a new memory is added, an LLM agent scans existing notes for semantic similarity and creates bidirectional links. Over time, a rich knowledge network emerges organically.

**Comparison to Mem0**: Where Mem0's graph memory relies on predefined schemas, A-MEM's connections emerge bottom-up. Mem0 is faster and more predictable; A-MEM builds richer knowledge networks.

**Benchmark results**: Demonstrated improvements on conversational QA benchmarks, accepted at NeurIPS 2025 poster session.

**Hermes relevance**: A-MEM's pattern is conceptually close to what an evolved LLM Wiki would look like with automatic cross-linking. The "notes that link themselves" pattern would be a natural extension of the existing wiki entity files.

### C. Hindsight: arXiv:2512.12818 (Dec 2025)

**Title**: "Hindsight is 20/20: Building Agent Memory that Retains, Recalls, and Reflects"  
A biomimetic memory architecture paper establishing the World/Experiences/Mental Models taxonomy and demonstrating LongMemEval SOTA at publication time.

### D. Multi-Layered Memory Architectures: arXiv:2603.29194 (Mar 2026)

**Title**: "Multi-Layered Memory Architectures for LLM Agents"

Formalizes the four-layer memory model for LLM agents:
- **Working memory** — In-context, ephemeral (the current conversation)
- **Episodic memory** — Specific past experiences, session summaries
- **Semantic memory** — General facts and user preferences
- **Procedural memory** — Skills, workflows, learned behaviors

Key finding: decompressing dialogue history across these layers with adaptive selection outperforms any single storage strategy.

**Direct Hermes application**: This framework maps directly to the Hermes memory stack:
- Working: Claude's context window
- Episodic: session search / conversation history
- Semantic: external memory provider (Mem0/Hindsight)
- Procedural: Skills system + wiki articles on how to do things

### E. VikingMem: arXiv:2605.29640 (VLDB 2026)

**Title**: "VikingMem: A Memory Base Management System for Stateful LLM-based Applications"  
Academic backing for OpenViking's filesystem paradigm. Accepted at VLDB 2026.

### F. Constitutional Memory Architecture: arXiv:2603.04740 (Mar 2026)

A four-layer governance hierarchy for memory-safe agents, with multi-layer semantic storage. Useful for agents that need audit trails.

### G. Episodic Memory is the Missing Piece: arXiv:2502.06975 (Feb 2025)

Argues that most current memory systems conflate episodic memory (specific events with temporal context) with semantic memory (general facts). Recommends separate episodic stores with time-indexed retrieval.

---

## 6. Supermemory

### What It Is

Supermemory is a memory and context platform claiming #1 on LongMemEval, LoCoMo, and ConvoMem (their own MemoryBench framework). Founded by the same team as Markprompt. Targets both developers (via API) and end-users (via app).

### Benchmark Claims: Verified?

| Benchmark | Supermemory Score | Notes |
|---|---|---|
| LongMemEval | **81.6%** | Self-reported; backbone unspecified |
| LoCoMo | **#1** | No % published on LoCoMo public page |
| ConvoMem | **#1** | No % published |

**Critical caveat**: Supermemory's #1 claims are on their own **MemoryBench** evaluation framework, not the official open-source benchmarks. Other providers (Mem0, Hindsight) do not accept Supermemory's rankings as authoritative.

Meanwhile, Mem0 v3 claims 94.8 on LongMemEval vs Supermemory's 81.6. If these were run on the same setup, Mem0 beats Supermemory by >13 points. The discrepancy suggests different backbone models, evaluation settings, or dates.

**Conclusion**: Supermemory's benchmark claims are contested. Their MemoryBench is useful as a standardized framework but their self-reported rankings should not be taken as absolute truth. As of June 2026, Mem0 v3 and Mastra Observational Memory both claim higher LongMemEval scores.

### Architecture

Supermemory's stack:
- **Memory Engine** — Extracts facts, tracks updates, resolves contradictions, auto-forgets expired info
- **User Profiles** — Auto-maintained: `profile.static` (long-term facts) + `profile.dynamic` (recent activity). One API call, ~50ms
- **Hybrid Search** — RAG + Memory in one query
- **Connectors** — Google Drive, Gmail, Notion, OneDrive, GitHub (real-time webhooks)
- **File Processing** — PDFs, images (OCR), videos (transcription), code (AST-aware chunking)

**Automatic forgetting**: Temporary facts ("I have an exam tomorrow") expire after the relevant date passes. Contradictions are resolved automatically. This is a key differentiator — most systems never forget.

### Self-Hosting: YES, but Complex

Supermemory supports:
- **Bare metal / Kubernetes self-host** — "Zero data leaves your perimeter"
- **VPC deployment** — AWS, GCP, or Azure inside your account
- **Hybrid** — Some components on-prem, some cloud

The self-host option exists but is **not documented** in the public README or docs as a simple docker-compose. It appears to be an enterprise offering requiring engagement with their team. No publicly documented one-command deploy exists as of June 2026.

The GitHub repo (`supermemoryai/supermemory`) contains the source code, so technically self-hostable from source, but there's no pre-built self-host container comparable to Hindsight's single-docker-run.

### Hermes Plugin

**There is a dedicated Hermes plugin**. Per the Supermemory README:
> "Supermemory comes built with Plugins for Claude Code, OpenCode, OpenClaw, and **Hermes**."

Plugin source: `https://github.com/NousResearch/hermes-agent` (the Supermemory memory provider for Hermes)

Tools available via plugin:
- `memory` — Save or forget information (auto-called during conversations)
- `recall` — Search memories + return user profile summary
- `context` — Inject full profile (preferences + recent activity) at session start

**Integration path**: MCP server at `https://mcp.supermemory.ai/mcp` with OAuth or API key. This is the cloud-only path. For self-host, would require deploying their stack and pointing to a local URL.

### Hermes Fit

- **Ready-to-use plugin** is compelling for rapid testing
- Automatic forgetting and profile system are mature features
- LongMemEval score (81.6%) is lower than Mem0 v3 and Hindsight
- Self-host situation is **unclear / enterprise-only** — cloud dependency is a concern
- Benchmark claims are contested
- **Recommendation**: Test the cloud plugin for rapid prototyping; evaluate self-host viability before committing

---

## 7. OpenViking

### What It Is

OpenViking is an open-source **Context Database** built by ByteDance's AI infrastructure team (volcengine org on GitHub). Academic backing: VikingMem paper (arXiv:2605.29640, VLDB 2026). License: **AGPLv3** (important — copyleft).

### Filesystem Paradigm

OpenViking's core insight: context management should work like a filesystem, not like flat vector search.

Everything maps to a `viking://` URI hierarchy:
```
viking://
├── resources/          ← Project docs, repos, web pages
│   └── my_project/
│       ├── docs/
│       └── src/
├── user/               ← Personal preferences, habits
│   └── memories/
│       ├── preferences/
│       └── habits/
└── agent/              ← Skills, instructions, task memories
    ├── skills/
    ├── memories/
    └── instructions/
```

Agent interacts with context via filesystem commands: `ls`, `find`, `tree`, `grep` — translated to retrieval operations.

### L0/L1/L2 Tiered Context Loading

Each piece of context is automatically processed into three layers:

| Level | Name | Content | Typical Size | Use |
|---|---|---|---|---|
| **L0** | Abstract | One-sentence summary | ~100 tokens | Quick relevance check, always loaded |
| **L1** | Overview | Core info + usage scenarios | ~2K tokens | Agent planning phase |
| **L2** | Details | Full original content | Full document | Deep dive, loaded on demand |

Every directory has its own `.abstract` (L0) and `.overview` (L1) files. The agent navigates top-down: check L0 to decide if a directory is relevant, load L1 for structure, fetch L2 only when needed.

**Token efficiency**: Only L0 summaries are loaded by default. Full documents pulled on demand. This is why OpenViking shows 63–91% token reduction in benchmarks.

### Directory Recursive Retrieval

1. **Intent analysis** — Query is decomposed into multiple retrieval conditions
2. **Initial positioning** — Vector retrieval finds the highest-score directory
3. **Refined exploration** — Secondary retrieval within that directory
4. **Recursive drill-down** — Recurses into subdirectories
5. **Result aggregation** — Merges all candidates

This "lock directory → refine content" strategy understands *context* (where information lives), not just *semantics* (what it says).

### Benchmark Performance

On LoCoMo (User Memory benchmark, May 2026):

| Agent | Without OpenViking | With OpenViking | Accuracy Δ | Latency Δ | Token Δ |
|---|---|---|---|---|---|
| OpenClaw | 24.20% | **82.08%** | +3.39× | -59% | **-91%** |
| **Hermes** | 33.38% | **82.86%** | +2.48× | -66% | -34% |
| Claude Code | 57.21% | **80.32%** | +1.40× | -58% | -63% |

The Hermes result is notable: 33% → 82% accuracy with 66% faster latency and 34% fewer tokens.

### Security and Jurisdiction Considerations

**Risk factors**:
- **Volcengine** is ByteDance's cloud infrastructure division — a Chinese technology company subject to PRC jurisdiction
- The default examples configure API calls to `ark.cn-beijing.volces.com` — Chinese cloud endpoints
- The VikingMem paper is from Zhejiang University + ByteDance researchers
- Community channels include WeChat and Lark groups

**Can it be run fully isolated?**

**Yes, with effort**:
1. OpenViking supports OpenAI as VLM provider (international endpoints only)
2. Supports Ollama for local models — fully air-gapped operation possible
3. The Python package and Rust CLI are open source (AGPLv3) — auditable
4. Storage is local filesystem — no Volcengine storage required
5. The server serves on `localhost:1933` — no outbound connections required if local models are used

**Isolation configuration**:
```json
{
  "vlm": { "provider": "openai", "model": "gpt-4o", "api_key": "...", "api_base": "https://api.openai.com/v1" },
  "embedding": { "dense": { "provider": "openai", "model": "text-embedding-3-large", "api_key": "...", "api_base": "https://api.openai.com/v1" } },
  "storage": { "workspace": "/home/hermes/openviking_workspace" }
}
```
Or fully local via Ollama. Once configured with non-Volcengine providers, there are no mandatory outbound calls to Chinese infrastructure.

**Remaining risks**:
- AGPLv3 license means any derivative work must also be AGPLv3 (relevant if Hermes ever becomes a product)
- The maintainers' primary use case is Volcengine deployment; non-Volcengine code paths may have more bugs
- Less community review of the non-Volcengine codepaths

**Recommendation**: OpenViking is technically isolatable, but the AGPLv3 license and Chinese-infra-first defaults create operational friction. For a personal agent with no commercial implications, the isolation is achievable. For a product, review carefully.

### Hermes Fit

- The L0/L1/L2 tiered loading aligns well with Hermes's context-budget concerns
- The filesystem paradigm maps to the existing `wiki/` directory structure
- The Hermes + OpenViking LoCoMo benchmark result (82.86%) is very promising
- **Key concern**: AGPLv3 license; Volcengine origin
- **Recommendation**: Strong candidate for the context-filesystem layer, especially if self-hosted with OpenAI or local models. Evaluate after Mem0/Hindsight prototype.

---

## 8. Graph Memory Approaches: Neo4j, Kuzu, Graphiti, Cognee

### The Graph Memory Premise

Graph memory stores entities as nodes and relationships as edges, with optional temporal annotations (valid_from, valid_until). Benefits:
- Multi-hop reasoning ("Who does Alice work with that also knows Bob?")
- Temporal fact tracking ("Alice worked at Google *from 2020–2024*, then joined Anthropic")
- Causal chains ("Event A caused Event B caused Event C")

### Current Approaches

**Graphiti (by Zep)**
- Core engine: open-source temporal knowledge graph for AI agents
- Storage backends: Neo4j (primary), FalkorDB, Kuzu (embedded), Amazon Neptune
- Key strength: tracks *when* facts changed, not just what they are
- Self-hosting: requires a running Graphiti server + a graph DB (Neo4j = heavy; Kuzu = lightweight)
- **Benchmark**: Zep (Graphiti-powered) scores 55–65% on LongMemEval depending on backbone
- **Commercial**: Zep Cloud wraps Graphiti for managed service

**Cognee**
- Open-source, focuses on turning existing corpuses into queryable knowledge graphs
- Graph backends: NetworkX (dev), Neo4j (prod), Kuzu (performance)
- Strength: document-to-graph ingestion; strong for structured knowledge extraction
- Limitation: weak on conversational/temporal memory vs Mem0
- **Benchmark**: Cognee competes with LightRAG on knowledge-base QA but lags on conversational memory

**Neo4j**
- Production-grade graph database, requires separate server process
- Heavy infrastructure for a personal agent
- Excellent for complex relationship queries at scale
- **At Hermes scale (~750 lines)**: overkill; operational cost exceeds benefit

**Kuzu**
- Embedded graph DB (like SQLite, but for graphs)
- No server process — runs in-process
- Excellent analytical performance
- **At Hermes scale**: appropriate technology, but requires Graphiti or Cognee as the layer on top

### Is Any Graph Approach Worth It for Hermes Now?

**Current scale assessment**:
- ~750 lines of memory content
- Single user (Isaac)
- Single agent (Hermes)
- Relationships are relatively simple (entities + their properties + temporal updates)

**Verdict**: **Not yet.** Graph databases add operational complexity (schema management, graph DB lifecycle, query language learning) without proportionate benefit at 750 lines.

The graph benefit emerges at ~5,000+ facts with rich cross-referencing. At current scale:
- A well-structured Markdown wiki with internal links provides 80% of the graph benefit
- Mem0's entity linking (built into v3) provides lightweight graph features without a separate DB
- Hindsight's entity/relationship indexing provides temporal graph features as part of its architecture

**Recommendation**: Use the implicit graph structure of the LLM Wiki (wikilinks between entities) + Mem0's entity linking. Revisit Graphiti/Kuzu when memory volume exceeds ~5,000 facts or when multi-hop reasoning failure rates are observed.

---

## Summary: Architecture Recommendations for Hermes

### Current Stack Assessment

The existing Hermes layered memory architecture is sound:
```
1. Live state      → Claude's context window (ephemeral)
2. Config          → CLAUDE.md, settings files
3. Boot docs       → Always-loaded context
4. Built-in memory → 2,200 chars (pointer/index only)
5. Session search  → Conversation history search
6. Skills          → Procedural knowledge
7. Markdown/wiki   → LLM Wiki at /claude-code-memory/wiki/
8. External        → Not yet active
```

### Recommended Enhancements

**Immediate (no new dependencies)**:
1. Expand the LLM Wiki with an **ingest loop**: when Hermes learns something durable, auto-create/update a wiki article
2. Add a **lint pass**: periodic cron to scan wiki articles for contradictions
3. Add **named queries** to the wiki index: structured query templates for common lookups

**Short-term prototype (pick one first)**:
- **Mem0 v3** — Best first choice: ADD-only algorithm is robust, self-hosted docker-compose is clean, community is large, Apache 2.0 license
- **Hindsight** — Second choice: biomimetic architecture is intellectually compelling, local Ollama support, but heavier

**Medium-term evaluate**:
- **Supermemory cloud** — Fast to test via dedicated Hermes plugin; questionable benchmark claims and cloud dependency
- **OpenViking** — Promising LoCoMo results with Hermes specifically; AGPLv3 and Volcengine origin require careful review

**Longer-term only if needed**:
- **Graphiti + Kuzu** — When multi-hop relationship reasoning fails at scale
- **A-MEM pattern** — Auto-linking wiki notes as the wiki grows; could be implemented within the existing wiki tooling

### Decision Matrix

| Provider | License | Self-host | LongMemEval | Effort | Hermes Fit |
|---|---|---|---|---|---|
| LLM Wiki (expand) | N/A | Yes (local) | N/A (different layer) | Low | ★★★★★ |
| Mem0 v3 | Apache 2.0 | Yes (docker-compose) | 94.8 (claimed) | Low | ★★★★☆ |
| Hindsight | MIT | Yes (single docker) | 91.4 (validated) | Low-Med | ★★★★☆ |
| Supermemory | Proprietary | Enterprise only | 81.6 | Low (cloud) | ★★★☆☆ |
| OpenViking | AGPLv3 | Yes (complex) | 82.86% (LoCoMo) | Med | ★★★☆☆ |
| Graphiti/Kuzu | MIT/Apache | Yes (heavy) | ~60% | High | ★★☆☆☆ |
| Cognee | Apache 2.0 | Yes | ~60–70% | High | ★★☆☆☆ |

### Final Recommendation

**Prototype Mem0 v3 first** (docker-compose, self-hosted, Anthropic backbone via LiteLLM), while continuing to expand the LLM Wiki. If Mem0's ADD-only approach proves too forgetful of contradictions over time, switch to Hindsight's biomimetic architecture. Keep OpenViking on the roadmap once the core semantic memory layer is stable.

The Karpathy LLM Wiki layer is already present and should be deepened (more ingest tooling, lint cron) regardless of which external provider is chosen — these are complementary, not competing layers.
