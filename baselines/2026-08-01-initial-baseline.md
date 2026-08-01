# Baseline — 2026-08-01 (initial capture)

Scope: know-hub document/search HTTP path, agent-hub CRAG reasoning path,
publish-services deployment time. See methodology at the bottom for exact
queries — re-run these unchanged for future captures so numbers are
comparable.

## Environment at capture time

| | |
|---|---|
| Stack | Local dev (`docker compose`, `infra/env/.env.dev`) |
| Stack uptime | ~1h |
| know-hub commit | `869588e` |
| agent-hub commit | `15b3c64` |
| agent-hub live governance (`GET /api/agent/governance`) | `top_k=6`, `cross_encoder_enabled=false`, `rerank_candidate_k=20`, `rerank_top_n=4`, `max_loops=2`, `max_regen=1`, `query_rewrite_enabled=true`, `grounding_enabled=true`, `generate_temperature=0.2` |

## Metrics

| # | Metric | Value | Source |
|---|---|---|---|
| 1 | know-hub `POST /api/documents` latency (p50/p95) | **No data — 0 samples** | Prometheus `knowhub_http_request_duration_seconds` |
| 2 | know-hub `GET /api/semantic/search` latency (p50/p95) | **No data — 0 samples** | Prometheus `knowhub_http_request_duration_seconds` |
| 3 | agent-hub CRAG per-node latency (retrieve/grade/rewrite/generate/ground-check) | **No data — 0 CRAG runs recorded** | Postgres `agent.agent_steps` |
| 4 | agent-hub loops/run, correction/abstention rate | 1 `INGESTION` run recorded (`SUCCEEDED`, `loops=9`); 0 `CRAG` runs | Postgres `agent.agent_runs` |
| 5 | Deployment time (`publish-services.yml` run duration) | **Not captured** | GitHub Actions |

## Gaps found during this capture

These are findings, not filled-in numbers — recorded here so the next capture attempt doesn't waste time rediscovering them:

1. **No real traffic on know-hub's document/search endpoints.** Prometheus only shows scrapes of `/metrics` and `/` — the upload and search paths have never been exercised in this environment, so metrics #1 and #2 have zero samples. Today's environment would produce a baseline of an *idle* service, not real usage.
2. **No CRAG chat runs recorded.** `agent.agent_runs` has zero rows with `kind='CRAG'`. This is the highest-value path from our plan and currently has no data at all.
3. **Suspicious zero latency on the one recorded ingestion run.** All 5 `agent.agent_steps` rows (`chunker`, `classifier`, `embedder`, `extractor`, `supervisor`) show `latency_ms = 0`. Sub-millisecond timing is implausible for an LLM-backed classifier/embedder step — this looks like a step-timing instrumentation gap (worth checking wherever `agent_steps` rows are inserted, e.g. `adapters/runs_repo.py`) rather than a real number. Don't trust step-latency numbers until this is confirmed either way.
4. **No GitHub Actions access from this machine.** `gh` CLI is not installed, and an unauthenticated REST call to `brainwave-project/brainwave` returned 404 (private repo). Deployment-time data needs `gh auth login` or a PAT before it's queryable.

## Next step to get a real v1 baseline

Generate a small amount of genuine traffic *through Kong* (not direct service calls, so latency reflects the real path — JWT validation, `X-User-Id` injection, etc.) before the next capture:

- 5–10 document uploads via `POST /api/documents`
- 5–10 semantic searches via `GET /api/semantic/search`
- 3–5 CRAG chat turns via `POST /api/agent/chat`

Then re-run the exact queries below and append a new dated section to this file (don't overwrite this one — baseline history should be append-only).

## Methodology (exact queries used)

**#1 / #2 — Prometheus (`http://localhost:9090`):**
```promql
histogram_quantile(0.50, sum(rate(knowhub_http_request_duration_seconds_bucket{path="/api/documents",method="POST"}[1h])) by (le))
histogram_quantile(0.95, sum(rate(knowhub_http_request_duration_seconds_bucket{path="/api/documents",method="POST"}[1h])) by (le))
# swap path="/api/semantic/search" for #2
```

**#3 — Postgres (`agent` schema):**
```sql
SELECT node,
       count(*) AS n,
       percentile_cont(0.5)  WITHIN GROUP (ORDER BY latency_ms) AS p50_ms,
       percentile_cont(0.95) WITHIN GROUP (ORDER BY latency_ms) AS p95_ms
FROM agent.agent_steps
GROUP BY node
ORDER BY node;
```

**#4 — Postgres:**
```sql
SELECT kind, status, count(*), avg(loops)::numeric(10,2), max(loops)
FROM agent.agent_runs
GROUP BY kind, status;
```

**#5 — GitHub Actions:**
```bash
gh run list --repo brainwave-project/brainwave --workflow=publish-services.yml --limit 10 \
  --json databaseId,createdAt,updatedAt,conclusion
# duration per run = updatedAt - createdAt
```
