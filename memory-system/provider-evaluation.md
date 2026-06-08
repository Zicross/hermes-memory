# External Memory Provider Evaluation

## Decision

No permanent provider selected yet. Prototype Honcho self-host first, then Mem0 v3, then Hindsight.

## Corrected benchmark numbers (June 8, 2026 research)

| Provider | LongMemEval | LoCoMo | Source |
|---|---|---|---|
| OMEGA | ~95.4% | — | self-reported |
| Mastra Observational Memory | 94.87% | — | self-reported |
| Mem0 v3 (April 2026) | **94.8%** | 91.6% | self-reported, Apache 2.0 |
| Hindsight (Gemini-3) | **91.4%** | — | independently validated (Virginia Tech) |
| ByteRover | 92.8% (LongMemEval-S) | 96.1% | arXiv:2604.01599 |
| RetainDB | 79% overall, 88% preference | — | self-reported |
| Supermemory | **81.6%** | #1 (no %) | self-reported on own MemoryBench |
| Honcho | Not published | Not published | honcho.dev/evals (unchecked) |
| OpenViking + Hermes | — | **82.86%** | Volcengine benchmarks (May 2026) |

**Critical correction**: The 96.1%/92.8% numbers in the initial handoff were misattributed to RetainDB. They belong to **ByteRover** (arXiv:2604.01599). RetainDB's own published numbers are 79%/88%.

**Benchmark caveats**: Different backbone LLMs, evaluation setups, and self-reporting make cross-provider comparison unreliable. Hindsight at 91.4% is the only independently validated number. All others should be treated as upper bounds with their specific backbone.

## Provider decision matrix

| Provider | License | Self-host | LongMemEval | Hermes plugin | Effort | Fit |
|---|---|---|---|---|---|---|
| LLM Wiki (expand) | N/A | Yes (local) | N/A | Built-in | Low | ★★★★★ |
| Honcho | AGPL-3.0 | docker compose | Not published | 5 tools + prefetch | Low | ★★★★★ |
| Mem0 v3 | Apache 2.0 | docker compose | 94.8% (claimed) | TBD | Low | ★★★★☆ |
| Hindsight | MIT | single docker | 91.4% (validated) | TBD | Low-Med | ★★★★☆ |
| ByteRover | Elastic 2.0 | Local CLI / cloud | 92.8% (paper) | Installed | Low-Med | ★★★☆☆ |
| Supermemory | Proprietary | Enterprise only | 81.6% | Installed | Low (cloud) | ★★★☆☆ |
| OpenViking | AGPL-3.0 | Complex | 82.86% (LoCoMo) | Installed | Med | ★★★☆☆ |
| RetainDB | BSL 1.1 / Apache | docker / npx | 79% | 10 tools + prefetch | Low-Med | ★★★☆☆ |
| Graphiti/Kuzu | MIT/Apache | Yes (heavy) | ~55–65% | None | High | ★★☆☆☆ |

## Prototype sequence

1. **Honcho self-host** — user/agent relationship modeling; mature Hermes plugin; docker compose + Anthropic API key
2. **Mem0 v3 self-host** — ADD-only fact store; Apache 2.0; docker compose; broader ecosystem
3. **Hindsight** — biomimetic learning; MIT; single docker run with Anthropic backend
4. **ByteRover or OpenViking** — if coding-project context trees or filesystem-paradigm context are needed

Do not enable multiple external providers simultaneously.
