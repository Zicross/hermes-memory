# Hindsight Implementation Note — 2026-06-08

## Status

Hindsight self-hosted is now active as Hermes default profile's external semantic memory provider.

## Working deployment

- Hermes config: `memory.provider: hindsight`
- Hermes plugin config: `~/.hermes/hindsight/config.json`
- Hindsight API: `http://127.0.0.1:8899`
- User systemd service: `~/.config/systemd/user/hindsight-api.service`
- Server venv: `/home/hermes/hindsight-venv`
- Local embedding/reranker model cache: `/home/hermes/.cache/hindsight-hf`
- Backend LLM: host Ollama `qwen3:8b` via `http://10.186.45.1:11434/v1`
- Dynamic bank template: `hermes-{profile}-{platform}-{user}`
- Default static bank fallback: `hermes-default`

Hindsight is assistive, not authoritative. Live config, `SYSTEM_BOOT.md`, `model-policy.md`, skills, curated docs, and direct tool outputs still win over semantic recall.

## Verification performed

```bash
hermes memory status
curl -sS http://127.0.0.1:8899/health
curl -sS http://127.0.0.1:8899/version
```

Results:

- `hermes memory status`: provider `hindsight`, plugin installed, status available.
- `/health`: `{"status":"healthy","database":"connected"}`
- `/version`: `{"api_version":"0.7.2", ...}`

Direct retain/recall smoke test passed using `hindsight_client.Hindsight` against bank `hermes-smoke-test`.

## Important pitfalls discovered

1. `127.0.0.1:8888` is SearXNG in this Hermes container. Do not use 8888 for Hindsight.
2. Use Hindsight on `127.0.0.1:8899`.
3. Hindsight's OpenAI-compatible Ollama base URL must include `/v1`: `http://10.186.45.1:11434/v1`.
4. Podman was not the right deployment route inside this LXD container:
   - `security.nesting=true` lets Podman initialize.
   - Rootless `slirp4netns` networking still fails.
   - Host-network Podman containers cannot make outbound sockets from inside this LXD (`Errno 13 Permission denied`).
   - DNS inside Podman also failed.
5. Direct Python venv + user systemd service works and is now the preferred route for this environment.
6. `hindsight-api-slim[all]` is large because it pulls Torch/CUDA packages; keep it isolated in `/home/hermes/hindsight-venv`, not Hermes' pipx venv.
7. `/healthz` can 404 on Hindsight; use `/health`, `/version`, `/openapi.json`, `/metrics`.

## Operational commands

```bash
systemctl --user status hindsight-api.service
systemctl --user restart hindsight-api.service
journalctl --user -u hindsight-api.service -n 120 --no-pager
curl -sS http://127.0.0.1:8899/health
curl -sS http://127.0.0.1:8899/version
hermes memory status
```

## Follow-up recall test

In a fresh Hermes session, test whether automatic Hindsight recall plus normal source verification can answer:

1. What is the current model/fallback/OpenRouter policy?
2. Where is the source of truth for model routing?
3. What did OpenClaw teach us about memory architecture?
4. Which memory provider was selected/deployed and why?
5. How is recall tested without hallucinating stale state?

Expected behavior: use Hindsight/session search as recall aids, then cite authoritative files/tool output rather than treating recalled facts as truth.
