---
title: External Memory Providers
created: 2026-06-08
updated: 2026-06-08
type: comparison
tags: [agent-memory, provider, comparison, semantic-memory]
sources: [raw/articles/honcho-readme.md, raw/articles/retaindb-readme.md, raw/articles/hindsight-readme.md, raw/articles/mem0-readme.md, raw/articles/supermemory-readme.md, raw/articles/openviking-readme.md, raw/articles/byterover-readme.md]
confidence: medium
contested: false
---

# External Memory Providers

## Current verdict

Do not select a permanent external provider yet. Build the markdown/source-of-truth and LLM Wiki layers first, then prototype [[honcho]] self-host and [[retaindb]] Local.

| Provider | Best use | Initial stance |
|---|---|---|
| [[honcho]] | User/agent/peer modeling | First self-host prototype |
| [[retaindb]] | Coding-agent/project memory | Second local prototype |
| [[hindsight]] | Learning-over-time service | Evaluate later; heavier |
| [[mem0]] | General memory baseline | Credible, but guard against black-box facts |
| [[supermemory]] | Hosted memory/context/connectors | Useful later if cloud dependency acceptable |
| [[openviking]] | Context filesystem | Interesting but not for sensitive core memory without review |
| [[byterover]] | Coding context tree | Evaluate after shared repo loop stabilizes |

Related: [[layered-agent-memory]], [[openclaw]]
