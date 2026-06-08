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
