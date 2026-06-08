---
type: reference
date: 2026-06-08
subject: Agent memory — the approaches and the philosophy behind each
audience: Hermes agent + future sessions (shared mental model)
related: ./scaling-and-multi-agent-memory.md, ./provider-comparison.md, ./production-patterns-and-recommendation.md, ./openclaw-cross-reference.md
---

# A primer on agent memory: the approaches and their philosophies

Every memory approach is a different answer to one problem, and most are quietly borrowing (or
rejecting) a model of *human* memory. That borrowed cognitive model is the "philosophy" of each.

## The one problem

An LLM is **stateless**: between turns it forgets everything except what is physically in its context
window. "Memory" is therefore not a model feature — it is any external scheme for deciding *what goes
back into the window, and when*. Every approach answers four questions:

1. **What is a "memory"?** a raw transcript / a distilled fact / an entity-relationship / a model of
   the user / a math vector?
2. **Who decides what to save and recall** — the agent (discretionary) or an automatic process?
3. **Where is it stored?** the window / files / a vector index / a graph / SQL / a holographic vector.
4. **How is the right piece retrieved at the right moment?**

## Two fault lines that explain most of the disagreement

- **Store vs. Process** — is memory *a place you put things* (a database) or *a process that curates*
  (extraction, reflection, consolidation)? The deepest split.
- **Discretionary vs. Automatic** — does the agent *choose* to remember/recall, or does it *happen to
  it*? This is the axis that decides whether memory survives scale + multiple agents.

---

## The schools (roughly "do nothing special" → "most exotic")

### 1. Context engineering — *"memory is attention, not storage"*
**Belief:** no separate memory problem, only a context-curation problem. Keep the smallest high-signal
token set in the window; compact when full. Memory ≈ disciplined working memory.
**Human analogy:** what you're actively holding in your head right now.
**Strong:** simplest, no infra, no retrieval errors, best accuracy *while history fits*.
**Breaks:** finite window; no persistence across days or agents.
**Who:** Anthropic "effective context engineering," Manus, the long-context camp.

### 2. Files / scratchpad — *"the filesystem is the brain"*
**Belief:** externalize memory like a notebook — durable, inspectable, editable, authored by the agent.
**Human analogy:** a lab notebook or personal wiki kept by hand.
**Strong:** transparent, unlimited, version-controllable, **fails loudly** not silently.
**Breaks:** **discretionary** (agent forgets to write/read), unbounded growth → context rot, no
concurrency control (parallel agents corrupt each other).
**Who:** CLAUDE.md, Cline Memory Bank, MemGPT self-edit, Anthropic memory tool. *The convergent
industry default — and the one whose ceiling we are hitting.*

### 3. Semantic / vector retrieval (RAG) — *"memory is associative recall by meaning"*
**Belief:** store everything as embeddings; recall = nearest-neighbor by meaning.
**Human analogy:** a smell triggering a related memory.
**Strong:** scales to large corpora; finds semantically related items keywords miss.
**Breaks:** retrieves *similar-not-relevant*; weak on multi-hop and time; can serve stale facts;
"lost in the middle." (This is the layer OpenClaw configured but never actually ran.)
**Who:** LangMem, raw vector DBs (Qdrant/Chroma/pgvector/LanceDB).

### 4. Automatic extraction & consolidation — *"memory should form itself, like sleep"*
**Belief:** don't store transcripts; a background process distills salient facts and reconciles
contradictions (ADD/UPDATE/DELETE) over time. Removes the agent's discretion. Modeled on memory
consolidation during sleep.
**Human analogy:** sleep deciding what to keep and merge.
**Strong:** agent *can't forget to save*; contradictions resolved; the write loop is the hard part
you don't have to rebuild.
**Breaks:** LLM cost per write; lossy/silent omissions; **false memories**; weak multi-fact recall.
**Who:** Mem0; ChatGPT "saved memories."

### 5. Structured / knowledge-graph, esp. *temporal* — *"memory is entities, relations, and time"*
**Belief:** knowledge has structure and truth changes; record entities + relationships + *when each
fact was true and when superseded* (bitemporal).
**Human analogy:** how you actually know a person — an updating web of facts, not past sentences.
**Strong:** multi-hop reasoning; "what changed since X"; **never serves stale state**; token-efficient.
**Breaks:** clean extraction is hard/expensive; needs a graph DB; brittle if extraction is wrong.
**Who:** Zep/Graphiti, Hindsight, HippoRAG.

### 6. User-modeling / theory-of-mind — *"model the person, don't log the conversation"*
**Belief:** the valuable memory is an evolving model of *who the user is* (beliefs, preferences,
psychology), continuously re-derived in the background.
**Human analogy:** a friend remembers *you*, not your exact sentences.
**Strong:** deep personalization; pre-computed insight, queryable in natural language.
**Breaks:** heavy (constant background reasoning); largely wasted on a single user; opaque;
eventually-consistent (model lags reality).
**Who:** Honcho.

### 7. Cognitive architecture (multi-store + reflection) — *"replicate human memory's structure"*
**Belief:** humans have working / episodic / semantic / procedural memory plus **reflection** that
turns experience into insight. Build those tiers explicitly.
**Human analogy:** the actual psychology of memory, modeled literally.
**Strong:** principled separation; reflection generates real learning.
**Breaks:** complex; **reflection is dangerous unsupervised** — token-expensive and degrades (OpenClaw's
nightly consolidation crons summarized its memory into word-salad).
**Who:** Generative Agents (memory stream + reflection), Letta tiers, Memori (conscious/auto modes).
**Variant — OS/paging:** treat context as RAM, external store as disk, and *page* memory in/out on
demand (MemGPT; OpenViking's L0→L1→L2 tiers). Same idea, computer metaphor.

### 8. Vector-symbolic / holographic — *"memory as algebra in high-dimensional space"*
**Belief:** store concepts as distributed vectors combined by math ops — *binding* ties a role to a
value (`city ⊗ Paris`), *bundling* superposes pairs into one fixed vector; recall = algebraic
*unbinding* + clean-up. The most biologically radical.
**Human analogy:** the brain as a hologram — every piece smeared across the whole.
**Strong:** compositional structured queries; noise-robust; zero dependencies.
**Breaks:** hard **capacity ceiling** — overload the fixed vector and it dissolves into noise.
Experimental.
**Who:** the "Holographic" (HRR) provider.

---

## How to hold it all together

**Store-centric** (files, vector, graph, holographic) believe intelligence is in *where/how you store*.
**Process-centric** (consolidation, user-modeling, reflection) believe it's in *the curation loop*.

The practical axis that matters most:
- **Discretionary** (context-engineering, files, raw RAG): predictable, inspectable, cheap — but the
  agent forgets to use it. → **predictability**.
- **Automatic** (Mem0, Honcho, graph extractors; and Hermes wraps providers to behave this way):
  the agent can't forget — but costlier, opaquer, invents false memories. → **intelligence**.

That trade — **predictability vs. intelligence** — is the real decision under everything.

**Three meta-truths:**
1. **Nobody uses just one.** Serious systems layer: a small file/context layer for stable truth + an
   automatic semantic/structured layer for episodic recall.
2. **They're all young.** None has solved memory-at-scale; the best vendors' numbers are self-reported.
3. **The store is rarely the bottleneck — the discipline is.** OpenClaw failed not because SQLite +
   keyword search is bad, but because writes were undisciplined and retrieval was never verified.
   Whatever philosophy you pick, the **write-routing policy and a recall test** make or break it.
