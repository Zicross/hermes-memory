# Hindsight operations after deployment — 2026-06-08

## Summary

Hindsight is now active as Hermes' external semantic memory provider, but it should be operated as one layer in a broader memory architecture rather than treated as a complete replacement for docs, skills, Obsidian, session search, or live verification.

## Verified current state

- Hermes memory provider: `hindsight`
- Hindsight API: `http://127.0.0.1:8899`
- Hindsight service: `hindsight-api.service`, user systemd, active/enabled
- Hindsight health: `/health` returns `{"status":"healthy","database":"connected"}`
- Hindsight version: `0.7.2`
- 8888 is **SearXNG**, not Hindsight
- Backend LLM: host Ollama `qwen3:8b` via `http://10.186.45.1:11434/v1`
- Working deployment: Python venv + user systemd, not Podman/Docker

## What changed for future Hermes sessions

`~/.hermes/hindsight/config.json` enables automatic memory behavior:

- `auto_recall: true` — Hindsight recall can be injected at turn start.
- `auto_retain: true` — conversation material can be retained for future semantic recall.
- `retain_async: true` — retention is background/asynchronous, so newly stated facts may not be immediately queryable.
- `bank_id_template: hermes-{profile}-{platform}-{user}` — memory is scoped by profile/platform/user.

Expected behavior:

- Future sessions should need fewer reminders about stable decisions, project context, preferences, and architecture facts.
- Recall may appear as injected memory context before the model responds.
- Hindsight recall is assistive, not authoritative; current-state claims still require live verification.

## Memory layer routing

| Need | Layer |
|---|---|
| Current truth: service status, ports, config, git status | Live tools/config |
| Compact durable fact/pointer needed in all sessions | Built-in memory |
| Reusable procedure, command sequence, pitfall, verification checklist | Skills |
| Exact conversation history | `session_search` |
| Fuzzy semantic recollection across conversations | Hindsight |
| Project decisions, architecture notes, roadmap, handoffs | Obsidian/Hermes Brain or `/home/hermes/claude-code-memory` |
| Shared engineering/project KB | `/home/hermes/claude-code-memory`, committed to GitHub |
| Periodic health/drift review | Cron/watchdog |

## Lessons learned

1. Hindsight is automatic semantic memory, not the full Hermes brain.
2. Obsidian/Hermes Brain is currently a curated filesystem wiki, not automatically fully indexed into Hindsight.
3. The shared memory repo is the Git-backed engineering/project memory layer.
4. Skills are the right place for repeatable operational procedures and pitfalls.
5. Built-in memory should stay compact and pointer-oriented.
6. Hindsight can recall stale or incomplete context; verify live state before making current claims.
7. Hindsight must run on 8899 in this container because 8888 is SearXNG.
8. The server venv has `hindsight_api` but may not include `hindsight_client`; use REST smoke tests unless the client package is explicitly installed.
9. `qwen3:8b` can emit startup verification empty-content warnings; actual retain/recall and `/health` are better indicators of operational health.
10. Podman became partially usable after LXD nesting, but nested networking failed; direct Python venv + user systemd is the correct deployment here.

## Operational checks

```bash
hermes memory status
systemctl --user is-active hindsight-api.service
systemctl --user is-enabled hindsight-api.service
curl -sS http://127.0.0.1:8899/health
curl -sS http://127.0.0.1:8899/version
ss -ltnp '( sport = :8888 or sport = :8899 )'
```

## Watchdog and audit cron

Two cron layers now exist:

### 1. Silent health watchdog

A silent script-only cron watchdog checks basic Hindsight availability:

- Script: `~/.hermes/scripts/hindsight-health-watchdog.py`
- Cron name: `Hindsight memory health watchdog`
- Job id: `1e2ab7582fae`
- Schedule: every 1 hour
- Delivery: origin chat
- Behavior: prints nothing when healthy; if the service or `/health` fails, it sends an alert with remediation commands.

Manual test:

```bash
~/.hermes/scripts/hindsight-health-watchdog.py
```

Expected healthy output: no output.

### 2. Agent-driven memory system audit

A second cron performs the higher-level audit Isaac requested: whether memory is actually working in the system and when to graduate toward Obsidian/internal-memory integration.

- Script: `~/.hermes/scripts/memory-system-audit-context.py`
- Cron name: `Hermes memory system audit and graduation review`
- Job id: `0ed72da710af`
- Schedule: every 24 hours
- Delivery: origin chat
- Attached skills: `hermes-memory-operations`, `hindsight-hermes-setup`, `obsidian`

The script collects grounded context only:

- Hindsight service, `/health`, `/version`, listeners, recent warnings
- Hermes memory provider/status
- Hindsight plugin config subset: `auto_recall`, `auto_retain`, `retain_async`, bank template
- REST retain/recall smoke test
- SYSTEM_BOOT memory pointers
- shared repo status and memory-system docs
- Obsidian vault discovery and candidate counts
- graduation criteria

The cron agent then emits a concise verdict:

- `HOLD` — not healthy enough or source-of-truth hygiene is off
- `READY_TO_DESIGN_BRIDGE` — Hindsight works; define Obsidian ingestion policy
- `READY_TO_PROTOTYPE_BRIDGE` — policy is clear; build selected-note ingestion
- `READY_TO_INDEX_SELECTED_NOTES` — prototype verified; safe to index approved notes

Important: the audit cron is report-only. It must not ingest Obsidian notes or rewrite memory directly.

## REST retain/recall smoke test

```bash
python3 - <<'PY'
import requests
base = 'http://127.0.0.1:8899'
bank = 'hermes-smoke-test'
fact = 'Smoke test fact: Hindsight listens on 127.0.0.1:8899 and 127.0.0.1:8888 is SearXNG.'

r = requests.post(
    f'{base}/v1/default/banks/{bank}/memories',
    json={
        'async': False,
        'items': [{
            'content': fact,
            'context': 'operational smoke test',
            'tags': ['smoke-test'],
        }],
    },
    timeout=300,
)
print('retain', r.status_code, r.text[:1000])
r.raise_for_status()

r = requests.post(
    f'{base}/v1/default/banks/{bank}/memories/recall',
    json={
        'query': 'Where does Hindsight listen, and what is port 8888?',
        'types': ['world', 'experience', 'observation'],
        'max_tokens': 1000,
        'budget': 'mid',
        'tags': ['smoke-test'],
        'tags_match': 'any_strict',
    },
    timeout=300,
)
print('recall', r.status_code, r.text[:4000])
r.raise_for_status()
PY
```

## Skill created

Created local skill:

- `hermes-memory-operations`

Purpose:

- Explain what Hindsight changes now that it is active.
- Route memory writes to the correct layer.
- Preserve post-task recording checklists.
- Avoid stale-source hallucinations.
- Verify service and recall health.

## Next recommended improvement

Add a deliberate bridge between Obsidian/Hermes Brain and Hindsight only after defining indexing policy. The current vault should not be blindly dumped into Hindsight. Prefer curated ingestion of stable notes with source paths/tags and a cron or manual command that indexes selected folders such as decisions, architecture, project context packets, and handoffs.
