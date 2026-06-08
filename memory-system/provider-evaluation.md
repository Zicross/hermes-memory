# External Memory Provider Evaluation

## Decision

Permanent provider selection is deferred. Prototype Honcho self-host first and RetainDB Local second.

## Shortlist

### Honcho

Best fit for Hermes user/agent modeling. Supports managed and self-hosted FastAPI. Hermes already has provider/plugin support.

### RetainDB Local

Best fit for local coding-agent/project context. Low-friction local API/viewer/MCP, BM25 + vector + graph retrieval, memory typing, consolidation, and reinforcement.

### Hindsight

Strong learning-over-time framing and self-hosted Docker. Heavier service; evaluate after Honcho/RetainDB if needed.

### Mem0

Mature universal memory layer with open-source/self-hosted option. Good baseline but may become black-box fact extraction without strict write policy.

### Supermemory

Strong cloud/product/connectors story and Hermes plugin support. Useful later for broad personal/company brain if cloud dependency is acceptable.

### OpenViking

Interesting context-filesystem model, but Volcengine/ByteDance origin makes it unsuitable as core memory for sensitive/China-related work without explicit review.

### ByteRover

Promising coding context tree/versioning/review workflow. Evaluate after the shared markdown repo proves its base loop.

## Raw stores

Qdrant/Chroma/LanceDB/pgvector/Neo4j/Kuzu are infrastructure, not memory systems. Use only if building custom retrieval after the policy/authority layer is stable.
